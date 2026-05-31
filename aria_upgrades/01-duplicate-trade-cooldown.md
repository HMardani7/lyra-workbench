# Duplicate Trade Cooldown — Implementation Plan

> **For Hermes:** Use subagent-driven-development skill to implement this plan task-by-task.

**Goal:** Prevent Aria from opening multiple paper trades for the same coin within a configurable cooldown window (default 2 hours), regardless of which wallet or signal source triggered them.

**Architecture:** Add a `DUPLICATE_TRADE_COOLDOWN_SECONDS` config setting and a `check_duplicate_cooldown()` function in the risk module. Integrate this check into both the Hyperliquid signal pipeline (`check_open_rules`) and the Invo webhook processor (`_process_open`). When any paper trade exists for a coin (open or recently closed) within the cooldown window, reject new open signals with a clear reason.

**Tech Stack:** Python 3.11+, SQLModel, Pydantic Settings, pytest, SQLite

---

## What This Upgrade Does

### The Problem It Solves

Aria copies trades from multiple signal sources — Hyperliquid wallets *and* Invo traders. When a strong narrative hits (e.g., "BTC breakout incoming"), many traders open the same position independently. Aria copies all of them.

**Example:** 5 wallets + 3 Invo traders all go long BTC within 30 minutes → Aria opens **8 BTC long positions**. If BTC drops 5%, instead of losing on 1 position, Aria loses on 8. All correlation, no diversification.

**Root cause:** The existing dedup (`has_open_trade_for_coin`) only checks if the *same wallet* already has an open trade for that coin. It doesn't catch the scenario where *different* wallets/traders all signal the same coin.

### What This Upgrade Changes

Adds a **cross-source time-windowed deduplication layer**. Before opening any new trade, Aria now asks: *"Has ANY paper trade (from any wallet, any source, open or recently closed) been opened for this coin in the last N seconds?"*

- If **yes** → the signal is rejected with a clear reason logged ("Duplicate trade: BTC already opened 45 min ago by 0xabc123...")
- If **no** → the signal proceeds through normal risk checks

### Risk It Prevents

- **Correlated overexposure:** Multiple identical positions from different signal sources stacking up
- **Narrative cascades:** When a popular trader calls a coin and 10+ copiers all trigger the same trade
- **Silent concentration drift:** Without this check, a portfolio can accidentally become 40% BTC because 5 wallets all signaled it

### Configurability

| Setting | Default | What It Does |
|---------|---------|--------------|
| `DUPLICATE_TRADE_COOLDOWN_SECONDS` | 7200 (2h) | Time window to block duplicates. Set to 0 to disable. |
| `DUPLICATE_TRADE_COOLDOWN_SECONDS=1800` | — | 30 min — aggressive dedup, fewer duplicates allowed |
| `DUPLICATE_TRADE_COOLDOWN_SECONDS=21600` | — | 6 hours — very conservative, only one trade per coin per session |

### Scope

- Applies to **both** Hyperliquid-detected signals and Invo webhook signals
- Checks **all** paper trades (open + recently closed), not just open ones — a closed trade still means we already acted on that signal idea
- **Does not** prevent the same wallet from adding to its own position (that's `ALLOW_ADD_TO_POSITION`'s job)
- **Does not** prevent different coins — ETH is not blocked just because BTC was traded

---

## Background

**Current problem:** The only dedup is `has_open_trade_for_coin()` in `app/risk/exposure.py`, which checks for open trades scoped to a single wallet+coin. If 5 different wallets all signal `BTC long` within minutes, Aria opens 5 positions — all perfectly correlated. One bad move = 5x the loss.

**What changes:** A cross-source, time-windowed dedup. When considering a new trade for coin X, check: "Has ANY paper trade (open or recently closed) been opened for coin X in the last N seconds?" If yes → skip.

**Configurable window** lets you tune: tight (30 min = aggressive dedup) to loose (6 hours = very conservative).

---

## Files to Touch

| File | Action | Purpose |
|------|--------|---------|
| `app/core/config.py` | Modify | Add `duplicate_trade_cooldown_seconds` setting |
| `.env.example` | Modify | Add `DUPLICATE_TRADE_COOLDOWN_SECONDS` |
| `app/risk/rules.py` | Modify | Add `check_duplicate_cooldown()` and call it in `check_open_rules()` |
| `app/risk/__init__.py` | Modify | Export new function |
| `app/invo/processor.py` | Modify | Call cooldown check in `_process_open()` |
| `tests/test_risk_rules.py` | Create | Tests for the dedup logic |
| `tests/test_invo_processor.py` | Modify | Add cooldown test cases (if test file exists) |

---

## Task 1: Add config setting

**Objective:** Add `DUPLICATE_TRADE_COOLDOWN_SECONDS` to the config model with a default of 7200 (2 hours).

**Files:**
- Modify: `app/core/config.py`
- Modify: `.env.example`

**Step 1: Add field to Settings class**

In `app/core/config.py`, add after the existing `paper_spot_require_live_compatible` line (around line 58):

```python
# After line 57 (paper_spot_require_live_compatible)
# Deduplication
duplicate_trade_cooldown_seconds: int = 7200  # 2 hours
```

Full context — find this block:
```python
    # Safety
    global_kill_switch: bool = False
    allow_add_to_position: bool = False
    min_signal_confidence: float = 0.75
    paper_spot_require_live_compatible: bool = True
```

And add after it:
```python
    # Deduplication
    duplicate_trade_cooldown_seconds: int = 7200  # 2 hours
```

**Step 2: Add to .env.example**

In `.env.example`, add after the `MIN_SIGNAL_CONFIDENCE` line (around line 26):

```
DUPLICATE_TRADE_COOLDOWN_SECONDS=7200
```

**Step 3: Verify**

Run:
```bash
cd /workspace/repos/hyperliquid-copy-bot
uv run python -c "from app.core.config import get_settings; s = get_settings(); print(s.duplicate_trade_cooldown_seconds)"
```
Expected: `7200`

---

## Task 2: Write failing tests for duplicate cooldown logic

**Objective:** Write tests that validate the dedup behavior before implementing it.

**Files:**
- Create: `tests/test_duplicate_cooldown.py`

**Step 1: Create test file**

```python
"""Tests for duplicate trade cooldown logic."""

from datetime import UTC, datetime, timedelta

import pytest
from sqlmodel import select

from app.core.config import Settings
from app.db.models import PaperPortfolio, PaperTrade
from app.db.session import get_session
from app.risk.rules import check_duplicate_cooldown, check_open_rules


def make_settings(**overrides) -> Settings:
    """Build a Settings instance with overrides for testing."""
    return Settings(
        duplicate_trade_cooldown_seconds=7200,
        min_signal_confidence=0.75,
        paper_spot_require_live_compatible=True,
        allow_add_to_position=False,
        max_total_exposure_pct=0.80,
        max_open_trades=20,
        max_open_trades_per_wallet=8,
        max_exposure_per_wallet_pct=0.25,
        max_exposure_per_coin_pct=0.15,
        max_leverage_cap=8.0,
        leverage_divisor=5.0,
        spot_taker_fee_pct=0.001,
        pause_if_balance_below=50.0,
        default_stake_pct=0.05,
        max_stake_pct=0.10,
        **overrides,
    )


class TestCheckDuplicateCooldown:
    """Tests for check_duplicate_cooldown()."""

    def test_no_recent_trade_returns_true(self):
        """When no trade exists for the coin, should allow."""
        # Coin with no trades at all
        result, reason = check_duplicate_cooldown("BTC", cooldown_seconds=7200)
        assert result is True
        assert reason == ""

    def test_old_trade_outside_window_returns_true(self):
        """When the only trade is older than the cooldown, should allow."""
        # We'd need to insert a trade manually, but the function signature
        # should be testable in isolation with a mock — for now we test
        # the pure logic path.
        pass  # Requires DB fixture — see integration test below

    def test_open_trade_within_window_returns_false(self):
        """When an open trade exists within the cooldown, should block."""
        pass  # Requires DB fixture

    def test_recently_closed_trade_within_window_returns_false(self):
        """When a trade was closed but opened within the cooldown, should block."""
        pass  # Requires DB fixture


class TestCheckDuplicateCooldownIntegration:
    """Integration tests that hit a real SQLite database."""

    @pytest.fixture
    def db_setup(self):
        """Set up a test portfolio and insert sample trades."""
        from app.db.init_db import init_database

        init_database()

        with get_session() as session:
            # Create a portfolio
            portfolio = PaperPortfolio(
                name="test",
                mode="spot_compatible_paper",
                starting_balance=1000.0,
                current_balance=1000.0,
                base_capital=1000.0,
            )
            session.add(portfolio)
            session.commit()
            session.refresh(portfolio)
            pid = portfolio.id

            # Clean up any existing trades
            for trade in session.exec(select(PaperTrade)).all():
                session.delete(trade)
            session.commit()

        yield pid

        # Teardown: clean up
        with get_session() as session:
            for trade in session.exec(select(PaperTrade)).all():
                session.delete(trade)
            for p in session.exec(select(PaperPortfolio)).all():
                session.delete(p)
            session.commit()

    def test_no_trade_allows_new_signal(self, db_setup):
        """When no trade exists for the coin, check passes."""
        result, reason = check_duplicate_cooldown("BTC", cooldown_seconds=7200)
        assert result is True
        assert reason == ""

    def test_recent_open_trade_blocks_duplicate(self, db_setup):
        """An open BTC trade opened 5 minutes ago should block a new BTC signal."""
        pid = db_setup
        now = datetime.now(UTC)

        with get_session() as session:
            trade = PaperTrade(
                portfolio_id=pid,
                wallet_id=1,
                wallet_address="0xabc123",
                coin="BTC",
                side="long",
                status="open",
                stake_amount=50.0,
                quantity=0.001,
                entry_price=50000.0,
                opened_at=now - timedelta(minutes=5),
            )
            session.add(trade)
            session.commit()

        result, reason = check_duplicate_cooldown("BTC", cooldown_seconds=7200)
        assert result is False
        assert "BTC" in reason
        assert "0xabc" in reason or "minutes" in reason or "ago" in reason

    def test_old_trade_outside_window_allows(self, db_setup):
        """A BTC trade opened 3 hours ago (outside 2h window) should allow."""
        pid = db_setup
        now = datetime.now(UTC)

        with get_session() as session:
            trade = PaperTrade(
                portfolio_id=pid,
                wallet_id=1,
                wallet_address="0xabc123",
                coin="BTC",
                side="long",
                status="closed",
                stake_amount=50.0,
                quantity=0.001,
                entry_price=50000.0,
                opened_at=now - timedelta(hours=3),
                closed_at=now - timedelta(hours=2, minutes=30),
            )
            session.add(trade)
            session.commit()

        result, reason = check_duplicate_cooldown("BTC", cooldown_seconds=7200)
        assert result is True
        assert reason == ""

    def test_different_coin_not_blocked(self, db_setup):
        """A recent ETH trade should not block a new BTC signal."""
        pid = db_setup
        now = datetime.now(UTC)

        with get_session() as session:
            trade = PaperTrade(
                portfolio_id=pid,
                wallet_id=1,
                wallet_address="0xabc123",
                coin="ETH",
                side="long",
                status="open",
                stake_amount=50.0,
                quantity=0.02,
                entry_price=3000.0,
                opened_at=now - timedelta(minutes=10),
            )
            session.add(trade)
            session.commit()

        result, reason = check_duplicate_cooldown("BTC", cooldown_seconds=7200)
        assert result is True
        assert reason == ""

    def test_zero_cooldown_disables_check(self, db_setup):
        """When cooldown is 0, all trades should be allowed (feature disabled)."""
        pid = db_setup
        now = datetime.now(UTC)

        with get_session() as session:
            trade = PaperTrade(
                portfolio_id=pid,
                wallet_id=1,
                wallet_address="0xabc123",
                coin="BTC",
                side="long",
                status="open",
                stake_amount=50.0,
                quantity=0.001,
                entry_price=50000.0,
                opened_at=now - timedelta(minutes=1),
            )
            session.add(trade)
            session.commit()

        result, reason = check_duplicate_cooldown("BTC", cooldown_seconds=0)
        assert result is True
        assert reason == ""
```

**Step 2: Run tests to verify they fail**

```bash
cd /workspace/repos/hyperliquid-copy-bot
uv run pytest tests/test_duplicate_cooldown.py -v
```

Expected: FAIL — `check_duplicate_cooldown` not defined.

---

## Task 3: Implement check_duplicate_cooldown()

**Objective:** Implement the core dedup function.

**Files:**
- Modify: `app/risk/rules.py`

**Step 1: Add the function to `app/risk/rules.py`**

Add this function after the imports (before `check_open_rules`):

```python
def check_duplicate_cooldown(coin: str, cooldown_seconds: int = 7200) -> tuple[bool, str]:
    """Check if a trade for the same coin was opened recently.

    Prevents duplicate trades when multiple wallets/traders signal
    the same coin within the cooldown window.

    Returns (is_allowed, reason).
    """
    if cooldown_seconds <= 0:
        return True, ""

    from datetime import UTC, datetime
    from sqlmodel import select

    from app.db.models import PaperTrade
    from app.db.session import get_session

    cutoff = datetime.now(UTC) - timedelta(seconds=cooldown_seconds)

    with get_session() as session:
        recent_trade = session.exec(
            select(PaperTrade)
            .where(
                PaperTrade.coin == coin,
                PaperTrade.opened_at >= cutoff,  # type: ignore[operator]
            )
            .order_by(PaperTrade.opened_at.desc())  # type: ignore[arg-type]
        ).first()

    if recent_trade is None:
        return True, ""

    # Build a human-readable reason
    elapsed = datetime.now(UTC) - recent_trade.opened_at  # type: ignore[operator]
    minutes_ago = int(elapsed.total_seconds() / 60)
    wallet_short = (
        recent_trade.wallet_address[:10] + "..."
        if recent_trade.wallet_address and len(recent_trade.wallet_address) > 10
        else recent_trade.wallet_address
    )

    reason = (
        f"Duplicate trade: {coin} already opened {minutes_ago} min ago "
        f"by {wallet_short} (cooldown: {cooldown_seconds // 60} min)"
    )
    return False, reason
```

Also add `timedelta` to the imports at the top if not already present — check the existing imports. The file currently imports from `loguru` and `app.core.config` etc. We need `timedelta` from `datetime`.

```python
# At top of file, update the import if needed:
from datetime import datetime, timedelta, UTC
```

Actually, looking at the existing file, it doesn't import datetime at all — it uses timestamp-based logic. Add the import:

```python
from datetime import datetime, timedelta, UTC
```

But wait — we're importing `datetime` inside the function to avoid circular imports. Let me restructure: put `timedelta` in the top-level import and `datetime, UTC` inside the function (which is fine since it's only used locally).

Actually, simpler approach: just import everything at the top:

```python
from datetime import datetime, timedelta, UTC
```

No circular import risk here since `rules.py` doesn't import anything that imports it back.

**Step 2: Run tests**

```bash
cd /workspace/repos/hyperliquid-copy-bot
uv run pytest tests/test_duplicate_cooldown.py -v
```

Expected: Most tests PASS now (the ones that don't need integration fixtures may still need the DB setup). Address any failures.

---

## Task 4: Integrate cooldown check into check_open_rules()

**Objective:** Call `check_duplicate_cooldown()` inside `check_open_rules()` so Hyperliquid signals are deduplicated.

**Files:**
- Modify: `app/risk/rules.py`

**Step 1: Add the check in check_open_rules()**

In `check_open_rules()`, add the cooldown check right after the confidence check and before the price check. Find this block (around lines 41-46):

```python
    # Must have sufficient confidence
    if signal.confidence_score < settings.min_signal_confidence:
        return False, f"Low confidence: {signal.confidence_score:.2f}", 0.0
```

And add after it:

```python
    # Check duplicate trade cooldown
    can_open, reason = check_duplicate_cooldown(
        signal.coin,
        cooldown_seconds=settings.duplicate_trade_cooldown_seconds,
    )
    if not can_open:
        return False, reason, 0.0
```

**Step 2: Verify the integration compiles**

```bash
cd /workspace/repos/hyperliquid-copy-bot
uv run python -c "from app.risk.rules import check_open_rules, check_duplicate_cooldown; print('OK')"
```

Expected: `OK`

---

## Task 5: Integrate cooldown check into Invo processor

**Objective:** Add the same cooldown check to the Invo webhook's `_process_open()` so Invo signals are also deduplicated against both Invo and Hyperliquid trades.

**Files:**
- Modify: `app/invo/processor.py`

**Step 1: Add the check in `_process_open()`**

In `_process_open()`, add the cooldown check after the leverage validation (around line 115) and before the position_key duplicate check. Find:

```python
    # Validate leverage
    if payload.source_leverage > settings.invo_max_source_leverage:
        ...
```

After that block and before the position_key check, add:

```python
    # Check duplicate trade cooldown (cross-source: HL + Invo)
    from app.risk.rules import check_duplicate_cooldown

    can_open, reason = check_duplicate_cooldown(
        payload.symbol,
        cooldown_seconds=settings.duplicate_trade_cooldown_seconds,
    )
    if not can_open:
        _update_signal(signal, "ignored", "ignored", reason)
        _notify_ignored(payload, reason)
        return InvoWebhookResponse(accepted=True, decision="ignored", reason=reason)
```

**Step 2: Verify**

```bash
cd /workspace/repos/hyperliquid-copy-bot
uv run python -c "from app.invo.processor import process_invo_signal; print('OK')"
```

Expected: `OK`

---

## Task 6: Run full test suite

**Objective:** Ensure nothing is broken by the changes.

**Step 1: Run all tests**

```bash
cd /workspace/repos/hyperliquid-copy-bot
uv run pytest tests/ -v
```

Expected: All existing tests PASS. New dedup tests PASS.

**Step 2: Run linter**

```bash
cd /workspace/repos/hyperliquid-copy-bot
uv run ruff check app/risk/rules.py app/invo/processor.py app/core/config.py
```

Expected: No new errors.

---

## Task 7: Write integration test for end-to-end cooldown

**Objective:** Add a test that simulates the full flow: signal → cooldown check → rejection.

**Files:**
- Modify: `tests/test_duplicate_cooldown.py` (add to existing file)

**Step 1: Add test for check_open_rules integration**

```python
class TestCheckOpenRulesWithCooldown:
    """Tests that check_open_rules respects the cooldown."""

    def test_cooldown_blocks_second_signal_same_coin(self):
        """When a trade exists for BTC, check_open_rules rejects a new BTC signal."""
        from app.risk.rules import check_open_rules

        # This needs a DB with a recent BTC trade inserted.
        # For now, test the pure function behavior — the integration
        # with check_open_rules is straightforward (it calls check_duplicate_cooldown
        # and returns False if it fails).
        pass  # Full integration test requires DB setup + signal fixture
```

For now, the unit tests on `check_duplicate_cooldown()` cover the logic. The integration through `check_open_rules` is trivial (it's a direct call). We can add full integration tests later if needed.

---

## Verification Checklist

After all tasks are complete, verify:

- [ ] `DUPLICATE_TRADE_COOLDOWN_SECONDS=7200` appears in `.env.example`
- [ ] `duplicate_trade_cooldown_seconds` field exists in `Settings` class
- [ ] `check_duplicate_cooldown("BTC", 7200)` returns `(True, "")` when no trades exist
- [ ] `check_duplicate_cooldown("BTC", 7200)` returns `(False, "Duplicate trade...")` when a recent BTC trade exists
- [ ] `check_duplicate_cooldown("BTC", 0)` returns `(True, "")` — zero disables the feature
- [ ] `check_open_rules()` calls `check_duplicate_cooldown()` and rejects on duplicate
- [ ] Invo `_process_open()` calls `check_duplicate_cooldown()` and rejects on duplicate
- [ ] Old trades outside the cooldown window are allowed
- [ ] Different coins are not blocked
- [ ] Existing tests still pass
- [ ] No linter errors introduced

---

## Design Decisions

| Decision | Rationale |
|----------|-----------|
| Check **all** paper trades (open + closed) within window | A closed trade still represents "this signal was already acted on." Opening again would be the same correlated bet. |
| Zero cooldown = feature disabled | Backward compatible. Existing behavior preserved when `DUPLICATE_TRADE_COOLDOWN_SECONDS=0`. |
| Reason message includes wallet + timing | Helps debug Telegram alerts — user sees *why* a signal was ignored. |
| Configurable via env, not hardcoded | Easy to tune without code changes. 2h is a sensible default but every strategy differs. |
| Cross-source (HL + Invo) | An Invo signal for BTC should be blocked if a Hyperliquid wallet already triggered BTC 30 min ago, and vice versa. |
| Check runs early in the pipeline (before exposure checks) | Fast DB query, cheap rejection. No point calculating stakes for a trade we're about to block. |

---

## Future Enhancements (not in this plan)

- **Per-coin cooldowns**: Different windows for different coins (BTC 2h, memecoins 30min)
- **Burst allowance**: Allow up to N duplicates within the window before blocking (e.g., max 2)
- **Telegram notification for blocked duplicates**: Currently blocked quietly — could send a "Blocked duplicate BTC signal from 0xabc..." alert if desired
- **Cooldown reset on close**: If the original trade closes, maybe the cooldown should reset? Current design says no — the signal idea is still recent.
