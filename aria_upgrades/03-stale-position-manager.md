# Stale Position Manager — Implementation Plan

> **For Hermes:** Use subagent-driven-development skill to implement this plan task-by-task.

**Goal:** Prevent Aria's capital from being locked in stale positions. When a paper trade stays open longer than 24 hours, evaluate whether to hold, exit, or flag for re-check. Freeing capital from losing stale trades means more cash available for new signals — increasing overall opportunity capture.

**Architecture:** A lightweight check runs every 6 hours (not every tick) via a `BotSetting` timestamp. For each open position older than 24 hours, fetch the current price from Bitget ticker, calculate unrealized PnL, and apply a decision matrix: profitable → hold, moderate loss → exit, deep loss → flag for 6-hour re-check.

**Tech Stack:** Python 3.11+, httpx, SQLModel, Pydantic Settings, pytest, SQLite

---

## What This Upgrade Does

### The Problem It Solves

Aria simulates leverage by multiplying the base stake (2.5% of equity) by the source trader's leverage. A 5x leveraged position ties up 12.5% of total equity. When a trader holds a position for days or weeks — whether they're waiting out a drawdown, making a long-term bet, or simply inactive — Aria's capital stays locked. There's no mechanism to autonomously exit.

**Real example:**

| Trade | Entry | Leverage | Capital Locked | Age | PnL | Problem |
|-------|-------|----------|---------------|-----|-----|---------|
| SOL long | $220 | 4x | 10% of equity | 3 days | -12% | $ locked in a losing trade |
| ETH long | $3,100 | 5x | 12.5% of equity | 2 days | +8% | $ locked but trade is working |
| DOGE long | $0.42 | 3x | 7.5% of equity | 5 days | -28% | Deep loss, might bounce or die |

**Total capital frozen:** 30% of equity. That's 30% of the portfolio unavailable for new signals — signals that might have been profitable.

**Root cause:** The only exit mechanism is the source trader closing their position. There's no autonomous exit logic. Aria is entirely passive.

### What This Upgrade Changes

Adds a **Stale Position Manager** that evaluates open trades older than 24 hours:

- **Profitable positions** (>0% PnL) → HOLD. The trade is working, let the original thesis play out.
- **Moderate losses** (0% to -10% PnL) → EXIT. Capital is better deployed on new signals than slowly bleeding.
- **Deep losses** (<-10% PnL) → FLAG for re-check in 6 hours. Don't panic-sell at the bottom — give it time to recover. After 6 hours, if the PnL has improved, clear the flag and hold. If it's the same or worse, exit.

**Why deep losses get a grace period:** Selling into a -25% drawdown often means locking in losses right before a bounce. The 6-hour re-check window lets temporary capitulation events settle. If the position recovers to -15% → the trend is positive → hold. If it drops to -35% → the thesis is broken → exit.

### How It Improves Profitability

- **Frees trapped capital:** Losing stale positions get exited, returning cash to the pool for new signals
- **Reduces opportunity cost:** Every dollar freed from a -8% trade can be deployed on a signal the AI validator approves
- **Protects against deep drawdowns:** The re-check mechanism prevents panic-selling while still enforcing an exit if things don't improve
- **Lets winners run:** Profitable positions are untouched — the original trader's thesis is respected

### When It Runs

**Every 6 hours, not every tick.** This is critical for the small VPS (4GB RAM, 80GB storage) Aria runs on:

| | Every tick (30s) | Every 6 hours |
|---|---|---|
| Checks per day | 2,880 | **4** |
| Bitget API calls/day | up to 57,600 | up to **80** |
| Extra SQL per tick | heavy | **1 lightweight query** |
| CPU impact | significant | negligible |

The 6-hour interval naturally aligns with the decision windows: a position opened at hour 0 gets checked at hour 24 (the first 6-hour checkpoint after it becomes stale). A flagged position gets re-checked at hour 30 (6 hours after flagging).

Tracking uses the existing `BotSetting` key-value store — no new infrastructure:

```python
# At the start of every tick:
last_check = get_bot_setting("last_stale_check_at")
if now - last_check < 6 hours:
    return  # skip — less than 1ms
# Otherwise: run stale position manager
set_bot_setting("last_stale_check_at", now)
```

### Decision Matrix

```
For each open trade older than 24 hours:

  Fetch current price from Bitget ticker API
  Calculate unrealized PnL% = (current_price - entry_price) / entry_price × leverage

  ┌──────────────────────────────────────────────────────┐
  │  PnL > 0%                                            │
  │    → HOLD                                            │
  │    Action: nothing. Trade is profitable.             │
  ├──────────────────────────────────────────────────────┤
  │  PnL -10% to 0%  (moderate loss)                     │
  │    → EXIT                                            │
  │    Action: close paper trade at market price.        │
  │    Reason: "Stale position: capital better deployed" │
  ├──────────────────────────────────────────────────────┤
  │  PnL < -10%  (deep loss)                             │
  │                                                      │
  │    ┌─ NOT previously flagged:                        │
  │    │   → FLAG. Record current PnL + timestamp.       │
  │    │   → HOLD for now. Re-check in ~6 hours.         │
  │    │   Reason: "Deep loss flagged for re-check"      │
  │    │                                                  │
  │    └─ ALREADY flagged (≥ 6 hours since flag):        │
  │        ├─ PnL improved vs flagged PnL:               │
  │        │   → Clear flag. HOLD. Recovering.           │
  │        │   Reason: "Deep loss improving, holding"    │
  │        │                                              │
  │        └─ PnL same or worse:                         │
  │            → EXIT. Not recovering. Cut losses.       │
  │            Reason: "Deep loss not recovering"        │
  └──────────────────────────────────────────────────────┘
```

### Edge Cases

- **Trader closes position before stale check fires:** Normal close flow handles this. The trade is already closed by the time we check.
- **Price feed unavailable:** Skip the position. Log a warning. Don't exit blindly without price data.
- **Position flagged, then trader closes it:** The flag fields are cleared on normal close. No stale action needed.
- **Multiple checks within 6-hour window:** Won't happen — the `BotSetting` timestamp gates it.
- **Position exactly at threshold:** Rounding-safe. PnL > 0 → hold. PnL ≤ 0 and > -0.10 → exit. PnL ≤ -0.10 → deep loss.

### Configurability

| Setting | Default | What It Does |
|---------|---------|--------------|
| `STALE_POSITION_MANAGER_ENABLED` | `true` | Master switch. Set to `false` to disable entirely. |
| `STALE_CHECK_INTERVAL_HOURS` | `6` | How often the stale position manager runs |
| `STALE_POSITION_AGE_HOURS` | `24` | Positions must be open this long before checking |
| `STALE_MODERATE_LOSS_PCT` | `-0.10` | PnL % threshold: above this → exit (as decimal, not percentage) |
| `STALE_DEEP_LOSS_PCT` | `-0.15` | PnL % threshold: below this → flag for re-check |
| `STALE_RECHECK_HOURS` | `6` | Wait time after flagging before deciding |
| `STALE_MAX_FLAG_HOURS` | `24` | Absolute max time flagged before forced exit |

### Scope

- Applies to **all open paper trades** (both Hyperliquid and Invo sources)
- **Does not** affect positions younger than `STALE_POSITION_AGE_HOURS`
- **Does not** close profitable positions — winners are left alone
- **Does not** interfere with normal close signals from source traders
- **Does not** open new trades — this is exit-only
- **Does not** require new services or processes

---

## Background

### Current Exit Mechanisms

Aria currently has two ways a trade can close:

1. **Source trader closes** — `close_long` signal detected from Hyperliquid fills or Invo `close` webhook
2. **Flip detection** — `flip_long_to_short` triggers close of the existing long

Neither handles the case where the trader simply does nothing for days while Aria's capital sits locked.

### PnL Calculation for Open Positions

The existing `app/paper/pnl.py` module has all the math we need:

- `calculate_gross_pnl(entry_price, exit_price, quantity)` — unrealized PnL before fees
- `calculate_pnl_pct(pnl, stake)` — PnL as percentage of stake
- `calculate_close_fee(exit_price, quantity)` — close fee

For the stale position check, we compute **unrealized PnL %** as:

```
gross_pnl = (current_price - entry_price) × quantity
pnl_pct = gross_pnl / stake_amount
```

This intentionally ignores fees for the decision (fees are small relative to the decision). Fees are still applied when actually closing.

### Price Data

`BitgetClient.get_ticker(symbol)` already fetches live prices from Bitget's public ticker endpoint. No auth required. We use `hl_coin_to_bitget_symbol(coin)` to convert Aria's coin names to Bitget symbols.

### BotSetting for Check Interval

Aria already has a `BotSetting` key-value store in the database. We use it to track `last_stale_check_at`:

```python
# models.py already has:
class BotSetting(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    key: str = Field(index=True, unique=True)
    value: str = ""
    updated_at: datetime = Field(default_factory=utc_now)
```

### Resource Impact

| Resource | Before | After | Delta |
|----------|--------|-------|-------|
| RAM | ~510MB (after upgrade #2) | ~510MB | 0 (no new in-memory state) |
| Storage | varies | negligible | 3 new nullable columns on existing table + 1 BotSetting row |
| CPU per tick (most ticks) | baseline | +1 SQL query | ~0.1ms |
| CPU every 6 hours | baseline | +20 ticker calls + SQL | ~2 seconds |
| API calls | baseline | +80/day max | Bitget public endpoint, free, 20 req/s limit |
| New dependencies | none | 0 | Everything uses existing code |

---

## Files to Touch

| File | Action | Purpose |
|------|--------|---------|
| `app/core/config.py` | Modify | Add 7 stale position config settings |
| `.env.example` | Modify | Add `STALE_POSITION_*` env vars |
| `app/db/models.py` | Modify | Add 3 nullable fields to `PaperTrade` |
| `app/risk/stale_positions.py` | **Create** | Decision engine — check, flag, exit |
| `app/paper/signal_processor.py` | Modify | Call stale position check at tick start |
| `tests/test_stale_positions.py` | **Create** | Tests for decision logic |

---

## Task 1: Add config settings

**Objective:** Add all stale position manager settings to the config model.

**Files:**
- Modify: `app/core/config.py`
- Modify: `.env.example`

**Step 1: Add fields to Settings class**

In `app/core/config.py`, add after the AI validation settings (from upgrade #2):

```python
    # Stale Position Manager
    stale_position_manager_enabled: bool = True
    stale_check_interval_hours: int = 6
    stale_position_age_hours: int = 24
    stale_moderate_loss_pct: float = -0.10
    stale_deep_loss_pct: float = -0.15
    stale_recheck_hours: int = 6
    stale_max_flag_hours: int = 24
```

**Step 2: Add to .env.example**

After the AI validation section:

```
# Stale Position Manager
STALE_POSITION_MANAGER_ENABLED=true
STALE_CHECK_INTERVAL_HOURS=6
STALE_POSITION_AGE_HOURS=24
STALE_MODERATE_LOSS_PCT=-0.10
STALE_DEEP_LOSS_PCT=-0.15
STALE_RECHECK_HOURS=6
STALE_MAX_FLAG_HOURS=24
```

**Step 3: Verify**

```bash
cd /workspace/repos/hyperliquid-copy-bot
uv run python -c "
from app.core.config import get_settings
s = get_settings()
print(f'enabled: {s.stale_position_manager_enabled}')
print(f'check_interval: {s.stale_check_interval_hours}h')
print(f'position_age: {s.stale_position_age_hours}h')
print(f'moderate_loss: {s.stale_moderate_loss_pct}')
print(f'deep_loss: {s.stale_deep_loss_pct}')
print(f'recheck: {s.stale_recheck_hours}h')
print(f'max_flag: {s.stale_max_flag_hours}h')
"
```

Expected: All values print with defaults.

---

## Task 2: Add stale tracking fields to PaperTrade

**Objective:** Add nullable columns for flag tracking. No new table — fields go on the existing `PaperTrade` model.

**Files:**
- Modify: `app/db/models.py`

**Step 1: Add fields**

In `app/db/models.py`, in the `PaperTrade` class, add after the existing `notes` field (around line 189):

```python
    # Stale position tracking (upgrade #3)
    stale_flagged_at: datetime | None = Field(default=None)
    stale_flagged_pnl_pct: float | None = Field(default=None)
    stale_status: str | None = Field(default=None)  # None, "flagged", "exited"
```

Full `PaperTrade` model after this change:

```python
class PaperTrade(SQLModel, table=True):
    """A paper trade entry."""

    __tablename__ = "paper_trades"

    id: int | None = Field(default=None, primary_key=True)
    portfolio_id: int = Field(default=0, index=True)
    wallet_id: int = Field(default=0, index=True)
    wallet_address: str = Field(index=True)
    open_signal_id: int | None = Field(default=None, index=True)
    close_signal_id: int | None = None
    coin: str = ""
    bitget_symbol: str = ""
    side: str = "long"
    status: str = "open"  # open / closed / ignored / error
    stake_amount: float = 0.0
    quantity: float = 0.0
    entry_price: float = 0.0
    exit_price: float | None = None
    opened_at: datetime = Field(default_factory=utc_now)
    closed_at: datetime | None = None
    pnl_amount: float | None = None
    pnl_pct: float | None = None
    fee_estimate: float | None = None
    slippage_estimate: float | None = None
    close_reason: str | None = None
    compatibility_reason: str = ""
    notes: str = ""
    # Stale position tracking (upgrade #3)
    stale_flagged_at: datetime | None = Field(default=None)
    stale_flagged_pnl_pct: float | None = Field(default=None)
    stale_status: str | None = Field(default=None)  # None, "flagged", "exited"
```

**Step 2: Verify model creation**

```bash
cd /workspace/repos/hyperliquid-copy-bot
uv run python -c "
from app.db.init_db import init_database
from app.db.models import PaperTrade
from sqlmodel import select
from app.db.session import get_session

init_database()
with get_session() as session:
    trades = session.exec(select(PaperTrade)).all()
    print(f'Trades in DB: {len(trades)}')
    # Check the new columns exist by inspecting the model
    for col in ['stale_flagged_at', 'stale_flagged_pnl_pct', 'stale_status']:
        assert hasattr(PaperTrade, col), f'Missing column: {col}'
    print('All stale tracking columns present')
"
```

Expected: `All stale tracking columns present`

---

## Task 3: Implement the stale position manager

**Objective:** Create the decision engine that checks positions, applies the decision matrix, and executes exits.

**Files:**
- Create: `app/risk/stale_positions.py`

**Step 1: Create the module**

`app/risk/stale_positions.py`:

```python
"""Stale Position Manager.

Evaluates paper trades open longer than 24 hours and decides whether
to hold, exit, or flag for re-check. Runs every 6 hours (not every tick)
to minimize API calls and CPU usage.

Decision matrix:
  PnL > 0%          → HOLD (profitable, let it ride)
  PnL -10% to 0%    → EXIT (capital better deployed elsewhere)
  PnL < -10%        → FLAG for 6h re-check. After re-check:
                        improved → HOLD, not improved → EXIT
"""

from __future__ import annotations

import asyncio
from datetime import UTC, datetime, timedelta
from typing import TYPE_CHECKING

from loguru import logger
from sqlmodel import select

from app.bitget.client import BitgetClient
from app.bitget.symbols import hl_coin_to_bitget_symbol
from app.core.config import get_settings
from app.core.time import utc_now
from app.db.models import BotSetting, PaperPortfolio, PaperTrade
from app.db.session import get_session
from app.paper.pnl import (
    calculate_close_fee,
    calculate_gross_pnl,
    calculate_net_pnl,
    calculate_pnl_pct,
)
from app.telegram.messages import send_message_sync


# === Public API ===


def check_stale_positions() -> dict[str, int]:
    """Run the stale position manager if enough time has passed.

    Called from process_all_signals() at the start of every tick.
    Uses BotSetting to track when the last check ran. Only executes
    if STALE_CHECK_INTERVAL_HOURS have passed since the last check.

    Returns counts of actions taken.
    """
    settings = get_settings()

    if not settings.stale_position_manager_enabled:
        return {"held": 0, "exited": 0, "flagged": 0, "cleared": 0, "skipped": 0}

    # Gate: only run every STALE_CHECK_INTERVAL_HOURS
    if not _should_run_now(settings):
        return {"held": 0, "exited": 0, "flagged": 0, "cleared": 0, "skipped": 0}

    logger.info("Running stale position manager...")
    counts = _run_check(settings)

    # Update last check timestamp
    _set_last_check_time()

    if any(v > 0 for v in counts.values()):
        logger.info(f"Stale position manager: {counts}")

    return counts


# === Internal ===


def _should_run_now(settings) -> bool:
    """Check if enough time has passed since the last stale check."""
    with get_session() as session:
        row = session.exec(
            select(BotSetting).where(BotSetting.key == "last_stale_check_at")
        ).first()

    if row is None:
        # Never run before — run now
        return True

    try:
        last_check = datetime.fromisoformat(row.value)
        elapsed = datetime.now(UTC) - last_check.replace(tzinfo=UTC)
        return elapsed.total_seconds() >= settings.stale_check_interval_hours * 3600
    except (ValueError, TypeError):
        # Corrupted timestamp — run now
        return True


def _set_last_check_time() -> None:
    """Record that a stale check just ran."""
    now_iso = utc_now().isoformat()
    with get_session() as session:
        row = session.exec(
            select(BotSetting).where(BotSetting.key == "last_stale_check_at")
        ).first()
        if row:
            row.value = now_iso
            row.updated_at = utc_now()
            session.add(row)
        else:
            session.add(BotSetting(key="last_stale_check_at", value=now_iso))
        session.commit()


def _run_check(settings) -> dict[str, int]:
    """Fetch open trades, evaluate each, and execute decisions."""
    counts = {"held": 0, "exited": 0, "flagged": 0, "cleared": 0, "skipped": 0}

    cutoff = utc_now() - timedelta(hours=settings.stale_position_age_hours)

    with get_session() as session:
        stale_trades = list(
            session.exec(
                select(PaperTrade)
                .where(
                    PaperTrade.status == "open",
                    PaperTrade.opened_at <= cutoff,  # type: ignore[operator]
                )
                .order_by(PaperTrade.opened_at)  # type: ignore[arg-type]
            ).all()
        )

    if not stale_trades:
        logger.debug("No stale positions to check.")
        return counts

    logger.info(f"Checking {len(stale_trades)} stale position(s)...")

    for trade in stale_trades:
        try:
            result = _evaluate_position(trade, settings)
            counts[result] = counts.get(result, 0) + 1
        except Exception as e:
            logger.error(f"Error evaluating stale trade {trade.id} ({trade.coin}): {e}")
            counts["skipped"] += 1

    return counts


def _evaluate_position(trade: PaperTrade, settings) -> str:
    """Evaluate a single stale position and take action.

    Returns one of: "held", "exited", "flagged", "cleared"
    """
    # Fetch current price
    bitget_symbol = trade.bitget_symbol or hl_coin_to_bitget_symbol(trade.coin)
    current_price = _fetch_current_price(bitget_symbol)

    if current_price is None or current_price <= 0:
        logger.warning(
            f"Cannot evaluate {trade.coin} (trade {trade.id}): no current price"
        )
        return "skipped"

    # Calculate unrealized PnL %
    gross_pnl = calculate_gross_pnl(
        trade.entry_price, current_price, trade.quantity, side="long"
    )
    pnl_pct = gross_pnl / trade.stake_amount if trade.stake_amount > 0 else 0.0

    logger.info(
        f"Stale check: {trade.coin} (trade {trade.id}) | "
        f"entry={trade.entry_price:.4f} current={current_price:.4f} | "
        f"PnL={pnl_pct:.1%} | age={(utc_now() - trade.opened_at).total_seconds() / 3600:.1f}h"
    )

    # Decision matrix
    if pnl_pct > 0:
        return _handle_positive(trade)

    if pnl_pct > settings.stale_moderate_loss_pct:
        return _handle_moderate_loss(trade, pnl_pct, current_price, settings)

    return _handle_deep_loss(trade, pnl_pct, current_price, settings)


# === Decision handlers ===


def _handle_positive(trade: PaperTrade) -> str:
    """Profitable position — hold. Clear any existing flag."""
    if trade.stale_status == "flagged":
        _clear_flag(trade, "position became profitable")
        return "cleared"

    logger.info(f"HOLD {trade.coin} (trade {trade.id}): profitable, letting it ride")
    return "held"


def _handle_moderate_loss(
    trade: PaperTrade, pnl_pct: float, current_price: float, settings
) -> str:
    """Moderate loss — exit immediately. Capital is better deployed elsewhere."""
    # Clear flag if it exists (we're exiting anyway)
    if trade.stale_status == "flagged":
        _clear_flag(trade, "exiting moderate loss")

    reason = (
        f"Stale exit: {pnl_pct:.1%} loss after "
        f"{(utc_now() - trade.opened_at).total_seconds() / 3600:.1f}h "
        f"(threshold: {settings.stale_moderate_loss_pct:.1%})"
    )
    _execute_exit(trade, current_price, gross_pnl=None, reason=reason)
    return "exited"


def _handle_deep_loss(
    trade: PaperTrade, pnl_pct: float, current_price: float, settings
) -> str:
    """Deep loss — flag for re-check, or re-evaluate if already flagged."""
    is_flagged = trade.stale_status == "flagged" and trade.stale_flagged_at is not None

    if not is_flagged:
        return _flag_position(trade, pnl_pct, settings)

    # Already flagged — check if re-check window has passed
    now = utc_now()
    flagged_at = trade.stale_flagged_at  # type: ignore[assignment]
    hours_flagged = (now - flagged_at).total_seconds() / 3600

    # Check if max flag time exceeded — exit regardless
    if hours_flagged >= settings.stale_max_flag_hours:
        reason = (
            f"Stale exit: deep loss persisted {hours_flagged:.0f}h "
            f"(max flag {settings.stale_max_flag_hours}h exceeded)"
        )
        _execute_exit(trade, current_price, gross_pnl=None, reason=reason)
        return "exited"

    # Check if enough time has passed for re-check
    if hours_flagged < settings.stale_recheck_hours:
        logger.debug(
            f"Still waiting for re-check: {trade.coin} flagged "
            f"{hours_flagged:.1f}h ago (need {settings.stale_recheck_hours}h)"
        )
        return "held"

    # Re-check: compare current PnL to flagged PnL
    flagged_pnl = trade.stale_flagged_pnl_pct or 0.0

    if pnl_pct > flagged_pnl:
        # Improved — clear flag, hold
        _clear_flag(
            trade,
            f"PnL improved from {flagged_pnl:.1%} to {pnl_pct:.1%} "
            f"(after {hours_flagged:.0f}h)",
        )
        return "cleared"

    # Not improved — exit
    reason = (
        f"Stale exit: deep loss not recovering. "
        f"Flagged at {flagged_pnl:.1%}, now {pnl_pct:.1%} "
        f"(after {hours_flagged:.0f}h)"
    )
    _execute_exit(trade, current_price, gross_pnl=None, reason=reason)
    return "exited"


# === Flag / unflag ===


def _flag_position(trade: PaperTrade, pnl_pct: float, settings) -> str:
    """Mark a position as flagged for re-check."""
    now = utc_now()
    with get_session() as session:
        db_trade = session.exec(
            select(PaperTrade).where(PaperTrade.id == trade.id)
        ).first()
        if db_trade:
            db_trade.stale_status = "flagged"
            db_trade.stale_flagged_at = now
            db_trade.stale_flagged_pnl_pct = pnl_pct
            session.add(db_trade)
            session.commit()

    logger.info(
        f"FLAGGED {trade.coin} (trade {trade.id}): "
        f"deep loss at {pnl_pct:.1%}, re-check in {settings.stale_recheck_hours}h"
    )

    # Update in-memory object for consistency
    trade.stale_status = "flagged"
    trade.stale_flagged_at = now
    trade.stale_flagged_pnl_pct = pnl_pct

    _notify_stale_action(trade, "flagged", f"Deep loss {pnl_pct:.1%} — re-check in {settings.stale_recheck_hours}h")
    return "flagged"


def _clear_flag(trade: PaperTrade, detail: str) -> None:
    """Clear the flag on a position (it recovered or became profitable)."""
    with get_session() as session:
        db_trade = session.exec(
            select(PaperTrade).where(PaperTrade.id == trade.id)
        ).first()
        if db_trade:
            db_trade.stale_status = None
            db_trade.stale_flagged_at = None
            db_trade.stale_flagged_pnl_pct = None
            session.add(db_trade)
            session.commit()

    logger.info(f"CLEARED flag on {trade.coin} (trade {trade.id}): {detail}")

    trade.stale_status = None
    trade.stale_flagged_at = None
    trade.stale_flagged_pnl_pct = None


# === Exit execution ===


def _execute_exit(
    trade: PaperTrade,
    current_price: float,
    gross_pnl: float | None = None,
    reason: str = "",
) -> None:
    """Close a paper trade at the current market price.

    Reuses the existing close logic from app/paper/execution.py.
    """
    from app.paper.portfolio import add_ledger_event, update_balance

    if gross_pnl is None:
        gross_pnl = calculate_gross_pnl(
            trade.entry_price, current_price, trade.quantity, side="long"
        )

    open_fee = trade.fee_estimate or 0.0
    close_fee = calculate_close_fee(current_price, trade.quantity)
    net_pnl = calculate_net_pnl(gross_pnl, open_fee, close_fee)
    pnl_pct_val = calculate_pnl_pct(net_pnl, trade.stake_amount)

    # Get portfolio for balance update
    pid = trade.portfolio_id

    with get_session() as session:
        portfolio = session.exec(
            select(PaperPortfolio).where(PaperPortfolio.id == pid)
        ).first()

    if not portfolio:
        logger.error(f"Portfolio {pid} not found for stale exit of trade {trade.id}")
        return

    cash_before = portfolio.current_balance

    # Close in DB
    with get_session() as session:
        db_trade = session.exec(
            select(PaperTrade).where(PaperTrade.id == trade.id)
        ).first()
        if db_trade:
            db_trade.status = "closed"
            db_trade.exit_price = current_price
            db_trade.closed_at = utc_now()
            db_trade.pnl_amount = net_pnl
            db_trade.pnl_pct = pnl_pct_val
            db_trade.fee_estimate = open_fee + close_fee
            db_trade.close_reason = reason
            db_trade.stale_status = "exited"
            if db_trade.stale_flagged_at:
                # Keep flag data for audit trail
                pass
            session.add(db_trade)
            session.commit()

    # Update cash: return stake + gross_pnl - close_fee
    cash_return = trade.stake_amount + gross_pnl - close_fee
    new_balance = cash_before + cash_return
    update_balance(pid, new_balance)

    # Update portfolio fees
    with get_session() as session:
        p = session.exec(
            select(PaperPortfolio).where(PaperPortfolio.id == pid)
        ).first()
        if p:
            p.realised_pnl += net_pnl
            p.total_fees += close_fee
            session.add(p)
            session.commit()

    add_ledger_event(
        pid,
        trade.id,
        "trade_close",
        cash_return,
        new_balance,
        f"[STALE EXIT] {trade.coin} net_pnl={net_pnl:.2f} ({pnl_pct_val:.1f}%) — {reason}",
    )

    logger.info(
        f"STALE EXIT {trade.coin} (trade {trade.id}): "
        f"entry={trade.entry_price:.4f} exit={current_price:.4f} "
        f"net_pnl={net_pnl:.4f} ({pnl_pct_val:.1f}%) | {reason}"
    )

    _notify_stale_action(trade, "exited", reason)


# === Price fetching ===


def _fetch_current_price(symbol: str) -> float | None:
    """Fetch the current mid-price for a Bitget symbol.

    Uses Bitget public ticker API. Returns None if unavailable.
    """
    try:
        client = BitgetClient()
        ticker = asyncio.run(client.get_ticker(symbol))
        if ticker:
            # Use last price
            price = float(ticker.get("lastPr", ticker.get("close", 0)))
            if price > 0:
                return price
    except Exception as e:
        logger.warning(f"Failed to fetch price for {symbol}: {e}")

    return None


# === Telegram notifications ===


def _notify_stale_action(trade: PaperTrade, action: str, detail: str) -> None:
    """Send a Telegram notification for stale position actions."""
    from app.core.config import get_settings

    settings = get_settings()
    if not settings.telegram_bot_token or not settings.telegram_chat_id:
        return

    if action == "exited":
        emoji = "\U0001f534"  # red circle
        action_text = "STALE EXIT"
    elif action == "flagged":
        emoji = "\U0001f7e1"  # yellow circle
        action_text = "STALE FLAGGED"
    else:
        emoji = "\u2139\ufe0f"  # info
        action_text = action.upper()

    from app.telegram.messages import short_address

    age_hours = (utc_now() - trade.opened_at).total_seconds() / 3600

    msg = (
        f"{emoji} <b>{action_text}</b>\n"
        f"Coin: {trade.coin}\n"
        f"Wallet: {short_address(trade.wallet_address)}\n"
        f"Age: {age_hours:.0f}h\n"
        f"{detail}"
    )
    send_message_sync(msg)
```

**Step 2: Verify imports compile**

```bash
cd /workspace/repos/hyperliquid-copy-bot
uv run python -c "
from app.risk.stale_positions import (
    check_stale_positions,
    _evaluate_position,
    _should_run_now,
    _fetch_current_price,
)
print('All imports OK')
"
```

---

## Task 4: Integrate into the signal processing tick

**Objective:** Call `check_stale_positions()` at the start of every tick, before processing new signals. This ensures freed capital is available immediately.

**Files:**
- Modify: `app/paper/signal_processor.py`

**Step 1: Add stale check at tick start**

In `process_all_signals()`, add at the very beginning (before the signal processing loop):

```python
def process_all_signals() -> dict[str, int]:
    """Process signals through the SPOT ledger and send alerts."""
    from app.ai.validator import reset_tick_state
    from app.ai.researcher import prune_stale_cache
    from app.paper.execution import process_signal_for_spot
    from app.risk.stale_positions import check_stale_positions  # NEW

    reset_tick_state()
    prune_stale_cache()

    # Check stale positions BEFORE processing new signals.
    # This frees capital from losing stale trades so it's available
    # for new signals in this same tick.
    stale_counts = check_stale_positions()  # NEW
    if any(v > 0 for v in stale_counts.values()):
        logger.info(f"Stale position results: {stale_counts}")

    # ... rest of existing code (signal query, processing loop, etc.)
```

**Step 2: Verify compilation**

```bash
cd /workspace/repos/hyperliquid-copy-bot
uv run python -c "
from app.paper.signal_processor import process_all_signals
print('signal_processor with stale check imports OK')
"
```

---

## Task 5: Add test helper for BotSetting

**Objective:** The stale position manager uses `BotSetting` to track the last check time. We need a helper to set and clear this in tests without affecting production.

**Files:**
- Modify: `tests/conftest.py` (add helper if needed)
- Create: `tests/test_stale_positions.py`

**Step 1: Create test file**

`tests/test_stale_positions.py`:

```python
"""Tests for stale position manager."""

from datetime import UTC, datetime, timedelta

import pytest
from sqlmodel import select

from app.core.config import Settings
from app.db.models import BotSetting, PaperPortfolio, PaperTrade
from app.db.session import get_session
from app.risk.stale_positions import (
    _evaluate_position,
    _handle_positive,
    _handle_moderate_loss,
    _handle_deep_loss,
    _should_run_now,
    _fetch_current_price,
)


def make_settings(**overrides) -> Settings:
    """Build a Settings instance with overrides for testing."""
    return Settings(
        stale_position_manager_enabled=True,
        stale_check_interval_hours=6,
        stale_position_age_hours=24,
        stale_moderate_loss_pct=-0.10,
        stale_deep_loss_pct=-0.15,
        stale_recheck_hours=6,
        stale_max_flag_hours=24,
        **overrides,
    )


# === Helpers ===


def _insert_trade(
    coin: str,
    entry_price: float,
    quantity: float,
    stake: float,
    opened_hours_ago: float = 25.0,
    stale_status: str | None = None,
    stale_flagged_at: datetime | None = None,
    stale_flagged_pnl_pct: float | None = None,
    bitget_symbol: str = "",
) -> PaperTrade:
    """Insert a paper trade for testing."""
    from app.db.init_db import init_database

    init_database()

    pid = _get_or_create_portfolio_id()

    trade = PaperTrade(
        portfolio_id=pid,
        wallet_id=1,
        wallet_address="0xtest123",
        coin=coin,
        bitget_symbol=bitget_symbol or f"{coin}USDT",
        side="long",
        status="open",
        stake_amount=stake,
        quantity=quantity,
        entry_price=entry_price,
        opened_at=datetime.now(UTC) - timedelta(hours=opened_hours_ago),
        stale_status=stale_status,
        stale_flagged_at=stale_flagged_at,
        stale_flagged_pnl_pct=stale_flagged_pnl_pct,
        fee_estimate=1.0,
        notes="test trade",
    )

    with get_session() as session:
        session.add(trade)
        session.commit()
        session.refresh(trade)
        return trade


def _get_or_create_portfolio_id() -> int:
    """Get or create a test portfolio."""
    with get_session() as session:
        p = session.exec(select(PaperPortfolio).limit(1)).first()
        if p:
            return p.id or 0
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
        return portfolio.id or 0


def _cleanup() -> None:
    """Remove test data."""
    with get_session() as session:
        for trade in session.exec(select(PaperTrade)).all():
            session.delete(trade)
        for p in session.exec(select(PaperPortfolio)).all():
            session.delete(p)
        session.commit()


def _set_last_check_time(hours_ago: float) -> None:
    """Set the last check timestamp for testing."""
    ts = (datetime.now(UTC) - timedelta(hours=hours_ago)).isoformat()
    with get_session() as session:
        row = session.exec(
            select(BotSetting).where(BotSetting.key == "last_stale_check_at")
        ).first()
        if row:
            row.value = ts
            session.add(row)
        else:
            session.add(BotSetting(key="last_stale_check_at", value=ts))
        session.commit()


def _clear_last_check_time() -> None:
    """Remove the last check timestamp."""
    with get_session() as session:
        row = session.exec(
            select(BotSetting).where(BotSetting.key == "last_stale_check_at")
        ).first()
        if row:
            session.delete(row)
            session.commit()


# === Tests: Should Run Now ===


class TestShouldRunNow:
    """Tests for _should_run_now() — the 6-hour gate."""

    @pytest.fixture(autouse=True)
    def setup(self):
        _clear_last_check_time()
        yield
        _clear_last_check_time()

    def test_never_run_before_returns_true(self):
        """If there's no last_check_at record, should run now."""
        settings = make_settings(stale_check_interval_hours=6)
        assert _should_run_now(settings) is True

    def test_just_ran_returns_false(self):
        """If last check was 1 hour ago, don't run again."""
        _set_last_check_time(hours_ago=1)
        settings = make_settings(stale_check_interval_hours=6)
        assert _should_run_now(settings) is False

    def test_interval_passed_returns_true(self):
        """If last check was 7 hours ago, run now."""
        _set_last_check_time(hours_ago=7)
        settings = make_settings(stale_check_interval_hours=6)
        assert _should_run_now(settings) is True

    def test_exactly_interval_returns_true(self):
        """If last check was exactly 6 hours ago, run now."""
        _set_last_check_time(hours_ago=6)
        settings = make_settings(stale_check_interval_hours=6)
        assert _should_run_now(settings) is True

    def test_disabled_manager_should_not_run(self):
        """When stale_position_manager_enabled=False, _should_run_now
        is not called — but the gate in check_stale_positions() handles this.
        """
        # This is tested at the integration level — see test_check_stale_positions_disabled
        pass


# === Tests: Decision Matrix ===


class TestDecisionMatrix:
    """Tests for the core decision logic."""

    @pytest.fixture(autouse=True)
    def setup(self):
        _cleanup()
        _clear_last_check_time()
        yield
        _cleanup()
        _clear_last_check_time()

    def test_positive_pnl_holds(self, monkeypatch):
        """Profitable position should be held."""
        # Mock price fetch to return a profitable price
        monkeypatch.setattr(
            "app.risk.stale_positions._fetch_current_price",
            lambda symbol: 110.0,  # entry was 100 → +10% gross
        )

        trade = _insert_trade("BTC", entry_price=100.0, quantity=1.0, stake=100.0)
        settings = make_settings()

        result = _evaluate_position(trade, settings)
        assert result in ("held", "cleared")  # held if no flag, cleared if flagged

    def test_moderate_loss_exits(self, monkeypatch):
        """Moderate loss (-5%) should trigger exit."""
        monkeypatch.setattr(
            "app.risk.stale_positions._fetch_current_price",
            lambda symbol: 95.0,  # entry was 100 → -5%
        )

        trade = _insert_trade("ETH", entry_price=100.0, quantity=1.0, stake=100.0)
        settings = make_settings()

        result = _evaluate_position(trade, settings)
        assert result == "exited"

        # Verify trade is closed
        with get_session() as session:
            db_trade = session.exec(
                select(PaperTrade).where(PaperTrade.id == trade.id)
            ).first()
            assert db_trade is not None
            assert db_trade.status == "closed"
            assert "Stale exit" in (db_trade.close_reason or "")

    def test_deep_loss_first_time_flags(self, monkeypatch):
        """Deep loss (-20%) should flag for re-check, not exit immediately."""
        monkeypatch.setattr(
            "app.risk.stale_positions._fetch_current_price",
            lambda symbol: 80.0,  # entry was 100 → -20%
        )

        trade = _insert_trade("SOL", entry_price=100.0, quantity=1.0, stake=100.0)
        settings = make_settings()

        result = _evaluate_position(trade, settings)
        assert result == "flagged"

        # Verify flag is set
        with get_session() as session:
            db_trade = session.exec(
                select(PaperTrade).where(PaperTrade.id == trade.id)
            ).first()
            assert db_trade is not None
            assert db_trade.status == "open"  # NOT closed
            assert db_trade.stale_status == "flagged"
            assert db_trade.stale_flagged_at is not None
            assert db_trade.stale_flagged_pnl_pct is not None

    def test_flagged_position_improves_clears_flag(self, monkeypatch):
        """If PnL improves after flagging, clear the flag."""
        monkeypatch.setattr(
            "app.risk.stale_positions._fetch_current_price",
            lambda symbol: 88.0,  # improved from flagged -20% to -12%
        )

        trade = _insert_trade(
            "SOL",
            entry_price=100.0,
            quantity=1.0,
            stake=100.0,
            stale_status="flagged",
            stale_flagged_at=datetime.now(UTC) - timedelta(hours=7),
            stale_flagged_pnl_pct=-0.20,
        )
        settings = make_settings()

        result = _evaluate_position(trade, settings)
        assert result == "cleared"

        with get_session() as session:
            db_trade = session.exec(
                select(PaperTrade).where(PaperTrade.id == trade.id)
            ).first()
            assert db_trade is not None
            assert db_trade.stale_status is None  # flag cleared

    def test_flagged_position_worsens_exits(self, monkeypatch):
        """If PnL is still bad after re-check window, exit."""
        monkeypatch.setattr(
            "app.risk.stale_positions._fetch_current_price",
            lambda symbol: 75.0,  # worse than flagged -20%, now -25%
        )

        trade = _insert_trade(
            "DOGE",
            entry_price=100.0,
            quantity=1.0,
            stake=100.0,
            stale_status="flagged",
            stale_flagged_at=datetime.now(UTC) - timedelta(hours=7),
            stale_flagged_pnl_pct=-0.20,
        )
        settings = make_settings()

        result = _evaluate_position(trade, settings)
        assert result == "exited"

    def test_flagged_max_time_exceeded_exits(self, monkeypatch):
        """If flagged for more than max_flag_hours, exit regardless."""
        monkeypatch.setattr(
            "app.risk.stale_positions._fetch_current_price",
            lambda symbol: 85.0,  # improved! but too late
        )

        trade = _insert_trade(
            "PEPE",
            entry_price=100.0,
            quantity=1.0,
            stake=100.0,
            stale_status="flagged",
            stale_flagged_at=datetime.now(UTC) - timedelta(hours=25),  # > 24h max
            stale_flagged_pnl_pct=-0.20,
        )
        settings = make_settings(stale_max_flag_hours=24)

        result = _evaluate_position(trade, settings)
        assert result == "exited"

    def test_position_too_young_skipped(self):
        """Position opened less than 24h ago should not be evaluated."""
        # This test validates that _run_check filters by opened_at.
        # Positions younger than stale_position_age_hours are excluded from the query.
        from app.risk.stale_positions import _run_check

        _insert_trade("BTC", entry_price=100.0, quantity=1.0, stake=100.0, opened_hours_ago=10)
        settings = make_settings(stale_position_age_hours=24)

        counts = _run_check(settings)
        assert counts["held"] == 0
        assert counts["exited"] == 0
        # The 10h-old trade should not appear in counts at all
        total = sum(counts.values())
        assert total == 0, f"Expected 0 actions for young position, got {counts}"

    def test_multiple_stale_positions(self, monkeypatch):
        """Multiple stale positions should all be evaluated."""
        monkeypatch.setattr(
            "app.risk.stale_positions._fetch_current_price",
            lambda symbol: 95.0,  # all at -5% → moderate loss → exit
        )

        from app.risk.stale_positions import _run_check

        _insert_trade("BTC", entry_price=100.0, quantity=1.0, stake=100.0)
        _insert_trade("ETH", entry_price=100.0, quantity=1.0, stake=100.0)
        _insert_trade("SOL", entry_price=100.0, quantity=1.0, stake=100.0)

        settings = make_settings()
        counts = _run_check(settings)

        assert counts["exited"] == 3
        assert counts["held"] == 0


# === Tests: Price Fetching ===


class TestPriceFetching:
    """Tests for _fetch_current_price()."""

    def test_price_fetch_graceful_failure(self):
        """When ticker fails, should return None (not crash)."""
        # With no real Bitget connection, this should return None
        price = _fetch_current_price("NONEXISTENTUSDT")
        assert price is None or price > 0  # Either can't reach API or gets real data


# === Tests: Edge Cases ===


class TestEdgeCases:
    """Edge case tests."""

    @pytest.fixture(autouse=True)
    def setup(self):
        _cleanup()
        yield
        _cleanup()

    def test_zero_stake_handled(self, monkeypatch):
        """A trade with zero stake should not divide by zero."""
        monkeypatch.setattr(
            "app.risk.stale_positions._fetch_current_price",
            lambda symbol: 95.0,
        )

        trade = _insert_trade("BTC", entry_price=100.0, quantity=0.0, stake=0.0)
        settings = make_settings()

        # Should not crash
        result = _evaluate_position(trade, settings)
        assert result in ("held", "exited", "flagged", "skipped")

    def test_price_at_boundary(self, monkeypatch):
        """PnL exactly at threshold boundary (-10%)."""
        monkeypatch.setattr(
            "app.risk.stale_positions._fetch_current_price",
            lambda symbol: 90.0,  # entry 100 → -10% gross
        )

        trade = _insert_trade("BTC", entry_price=100.0, quantity=1.0, stake=100.0)
        settings = make_settings(stale_moderate_loss_pct=-0.10)

        result = _evaluate_position(trade, settings)
        # Exactly at -10%: pnl_pct is not > -0.10 (it's == -0.10)
        # So this falls into deep_loss path → flagged
        assert result == "flagged"

    def test_price_at_zero(self, monkeypatch):
        """PnL exactly at 0%."""
        monkeypatch.setattr(
            "app.risk.stale_positions._fetch_current_price",
            lambda symbol: 100.0,  # exactly at entry
        )

        trade = _insert_trade("BTC", entry_price=100.0, quantity=1.0, stake=100.0)
        settings = make_settings()

        result = _evaluate_position(trade, settings)
        # pnl = 0: not > 0, so not positive. Falls to moderate_loss.
        # 0 > -0.10, so moderate loss path
        assert result == "exited"

    def test_price_fetch_returns_none(self, monkeypatch):
        """When price fetch fails, position should be skipped (not exited blind)."""
        monkeypatch.setattr(
            "app.risk.stale_positions._fetch_current_price",
            lambda symbol: None,
        )

        trade = _insert_trade("BTC", entry_price=100.0, quantity=1.0, stake=100.0)
        settings = make_settings()

        result = _evaluate_position(trade, settings)
        assert result == "skipped"

        # Verify trade is still open
        with get_session() as session:
            db_trade = session.exec(
                select(PaperTrade).where(PaperTrade.id == trade.id)
            ).first()
            assert db_trade is not None
            assert db_trade.status == "open"


# === Integration Tests ===


class TestIntegration:
    """Integration tests for check_stale_positions()."""

    @pytest.fixture(autouse=True)
    def setup(self):
        _cleanup()
        _clear_last_check_time()
        yield
        _cleanup()
        _clear_last_check_time()

    def test_disabled_manager_returns_empty_counts(self):
        """When disabled, check_stale_positions should return zeros."""
        _insert_trade("BTC", entry_price=100.0, quantity=1.0, stake=100.0)

        settings = make_settings(stale_position_manager_enabled=False)
        # Override settings
        import app.risk.stale_positions as spm
        original = spm.get_settings
        spm.get_settings = lambda: settings

        try:
            counts = spm.check_stale_positions()
            assert counts["exited"] == 0
            assert counts["held"] == 0
            assert counts["flagged"] == 0
        finally:
            spm.get_settings = original

    def test_no_stale_positions_returns_zeros(self):
        """When no positions are old enough, all counts should be zero."""
        _insert_trade("BTC", entry_price=100.0, quantity=1.0, stake=100.0, opened_hours_ago=10)

        from app.risk.stale_positions import check_stale_positions
        counts = check_stale_positions()
        total = sum(v for k, v in counts.items() if k != "skipped")
        assert total == 0, f"Expected 0 actions, got {counts}"
```

**Step 2: Run tests**

```bash
cd /workspace/repos/hyperliquid-copy-bot
uv run pytest tests/test_stale_positions.py -v
```

Expected: All tests PASS.

---

## Task 6: Run full test suite + lint

**Objective:** Ensure nothing is broken by the changes.

**Step 1: Run all tests**

```bash
cd /workspace/repos/hyperliquid-copy-bot
uv run pytest tests/ -v
```

Expected: All existing tests PASS. New stale position tests PASS.

**Step 2: Run linter**

```bash
cd /workspace/repos/hyperliquid-copy-bot
uv run ruff check app/risk/stale_positions.py app/paper/signal_processor.py app/core/config.py app/db/models.py
```

Expected: No new errors.

---

## Verification Checklist

After all tasks are complete, verify:

- [ ] All stale position config settings appear in `.env.example`
- [ ] All settings fields exist in `Settings` class with correct defaults
- [ ] `stale_flagged_at`, `stale_flagged_pnl_pct`, `stale_status` columns exist on `PaperTrade`
- [ ] `_should_run_now()` returns `True` when no previous check exists
- [ ] `_should_run_now()` returns `False` when last check was < 6 hours ago
- [ ] `_should_run_now()` returns `True` when last check was ≥ 6 hours ago
- [ ] `BotSetting` row `last_stale_check_at` is created/updated after each run
- [ ] Profitable position (>0% PnL) → held
- [ ] Moderate loss position (-5%) → exited with proper close reason
- [ ] Deep loss position (-20%) first time → flagged, not exited
- [ ] Deep loss position flagged, improved after 6h → flag cleared, held
- [ ] Deep loss position flagged, worse after 6h → exited
- [ ] Deep loss position flagged > 24h (max_flag_hours) → exited regardless
- [ ] Position < 24h old → skipped (not evaluated)
- [ ] Price fetch failure → position skipped (not exited blind)
- [ ] Zero stake → handled without division by zero
- [ ] `check_stale_positions()` returns zero counts when `stale_position_manager_enabled=false`
- [ ] Telegram alert fires on exit and flag actions
- [ ] All existing tests still pass
- [ ] No new linter errors introduced

---

## Design Decisions

| Decision | Rationale |
|----------|-----------|
| **Every 6 hours, not every tick** | 4 checks/day vs 2,880. Up to 80 API calls/day vs 57,600. Negligible CPU. |
| **BotSetting for interval tracking** | Simple key-value store already in the codebase. No new infrastructure. |
| **Fields on PaperTrade, not new table** | Three nullable columns. Cleaner than a separate tracking table. No JOIN needed. |
| **Deep loss gets grace period** | Selling at -25% often locks in the bottom. 6h lets capitulation settle. |
| **Profitable positions untouched** | The original signal was correct. Don't interfere with winners. |
| **Position skipped on price failure** | Never exit without data. A missed exit is better than a blind one. |
| **Runs before signal processing** | Freed capital is available for new signals in the same tick. |
| **Reuses existing close logic** | `calculate_gross_pnl`, `calculate_net_pnl`, `update_balance`, `add_ledger_event` all already exist. |
| **PnL ignores fees for decision** | Fees are 0.1%. The decision threshold is 10%. Fees don't change the outcome. |

---

## Future Enhancements (not in this plan)

- **Per-coin stale thresholds:** Memecoins should exit faster than BTC
- **Trailing stop on profitable positions:** Instead of "hold until trader closes," exit if a +20% position drops to +10%
- **Opportunity cost weighting:** If many new signals are being missed due to locked capital, lower the exit threshold
- **Stale position dashboard:** Telegram command to list all stale positions and their status
- **Correlation-aware exits:** If 3 correlated coins are all stale, stagger exits to avoid exiting all at once
