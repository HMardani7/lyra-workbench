# Automated Stop-Loss — Implementation Plan

> **For Hermes:** Use subagent-driven-development skill to implement this plan task-by-task.

**Goal:** Protect Aria from outsized losses. Add a hard stop-loss that monitors open positions every tick and exits when a position drops below a configurable loss threshold. Optional trailing stop that ratchets up as the position profits.

**Architecture:** A `check_stop_losses()` function runs in `process_all_signals()` (every tick). For each open position, fetch the current price from Bitget's public ticker, compute unrealized PnL%, and compare against the stop-loss threshold. If breached → close at market. All exits logged with reason `stop_loss`. A `trailing_stop_high` field on `PaperTrade` tracks the highest price seen (for trailing stop).

**Tech Stack:** Python 3.11+, httpx, SQLModel, Pydantic Settings, pytest, SQLite

---

## What This Upgrade Does

### The Problem It Solves

Your avg loss (-$5.84) is 2.4× your avg win (+$2.42). The stale position manager catches leaking positions after 24 hours — but by then a -5% trade can be -20%. Real example from your data:

| Trade | Entry | Current Loss | Stale Check? |
|-------|-------|-------------|--------------|
| XRP | $1.27 | exited at -4.9% after 13h | Caught by Invo close, not stale |
| HYPE | $73.14 | exited at -4.4% after 11h | Same — source trader closed |

If the source trader had held XRP to -20%, Aria would have held too. The stale manager would catch it... **24 hours later**. That's $40 of losses on a $200 stake — when a -15% stop would have cut it at $30.

### What This Upgrade Changes

Adds a **hard stop-loss** + optional **trailing stop** that runs every tick:

**Hard stop:** Position PnL drops below `STOP_LOSS_PCT` → close immediately.

```
Example: BTC entry $70,000, STOP_LOSS_PCT = -15%
  Price drops to $59,500 → PnL = -15% → close at market
```

**Trailing stop (optional):** Tracks the highest price seen. Stop triggers when price drops X% from that high.

```
Example: BTC entry $70,000, TRAILING_STOP_PCT = -10%
  Price rises to $75,000 → trailing_high = $75,000
  Price drops to $67,500 (-10% from $75,000) → close
  (Position was up $5,000, now locking in $2,500 profit)
```

Trailing stop always uses the **better** of the hard stop and the trailing stop — whichever triggers first at a higher price.

### Decision: Hard Stop vs Trailing Stop

| Type | Default | Use Case |
|------|---------|----------|
| Hard stop | `STOP_LOSS_PCT = -0.15` | Always on. Catches positions that never went profitable. |
| Trailing stop | `TRAILING_STOP_ENABLED = true` | Protects profits on winning positions. |
| Trailing distance | `TRAILING_STOP_PCT = -0.10` | 10% drop from peak triggers exit. |

If both are enabled, the position exits at `max(hard_stop_price, trailing_stop_price)` — whichever gives the better exit.

### Configurability

| Setting | Default | What It Does |
|---------|---------|--------------|
| `STOP_LOSS_ENABLED` | `true` | Master switch for hard stop-loss |
| `STOP_LOSS_PCT` | `-0.15` | Close when unrealized PnL ≤ -15% |
| `TRAILING_STOP_ENABLED` | `true` | Enable trailing stop |
| `TRAILING_STOP_PCT` | `-0.10` | Close when price drops 10% from peak |
| `TRAILING_STOP_ACTIVATION_PCT` | `0.05` | Only start trailing after position is +5% |

### Scope

- Applies to **all open paper trades** (Invo source only, after Hyperliquid removal)
- Monitored **every tick** (~30 seconds) — uses Bitget public ticker (free, no auth)
- Ticker calls are batched: one call fetches all symbol prices
- Positions younger than 2 minutes are skipped (avoids stop-loss on brief entry volatility)
- Does NOT interfere with normal close signals from Invo
- Close reason is logged as `"stop_loss"` or `"trailing_stop"`

---

## Files to Touch

| File | Action | Purpose |
|------|--------|---------|
| `app/core/config.py` | Modify | Add stop-loss config settings |
| `.env.example` | Modify | Add `STOP_LOSS_*` / `TRAILING_STOP_*` env vars |
| `app/db/models.py` | Modify | Add `trailing_stop_high` to `PaperTrade` |
| `app/risk/stop_loss.py` | **Create** | Stop-loss checker + trailing stop logic |
| `app/paper/signal_processor.py` | Modify | Call `check_stop_losses()` in `process_all_signals()` |
| `tests/test_stop_loss.py` | **Create** | Tests for stop-loss logic |

---

## Task 1: Add config and DB fields

**File:** `app/core/config.py`

```python
    # Stop-Loss
    stop_loss_enabled: bool = True
    stop_loss_pct: float = -0.15
    trailing_stop_enabled: bool = True
    trailing_stop_pct: float = -0.10
    trailing_stop_activation_pct: float = 0.05
```

**File:** `app/db/models.py`

In `PaperTrade`, add after the stale tracking fields:

```python
    # Stop-loss tracking (upgrade #4)
    trailing_stop_high: float | None = Field(default=None)
```

Add migration in `init_db.py`:

```python
    if "paper_trades" in inspector.get_table_names():
        existing = {c["name"] for c in inspector.get_columns("paper_trades")}
        if "trailing_stop_high" not in existing:
            with engine.connect() as conn:
                conn.execute(
                    text("ALTER TABLE paper_trades ADD COLUMN trailing_stop_high FLOAT")
                )
                conn.commit()
            logger.info("Migration: added column paper_trades.trailing_stop_high")
```

---

## Task 2: Build stop-loss engine

**File:** `app/risk/stop_loss.py`

```python
"""Stop-loss monitor — checks open positions every tick for exit conditions."""

from __future__ import annotations

from datetime import UTC, datetime, timedelta

from loguru import logger
from sqlmodel import select

from app.bitget.client import BitgetClient
from app.bitget.symbols import hl_coin_to_bitget_symbol
from app.core.config import get_settings
from app.db.models import PaperTrade
from app.db.session import get_session


def check_stop_losses() -> dict[str, int]:
    """Check all open positions against stop-loss thresholds.

    Called from process_all_signals() every tick.
    Fetches all current prices in one batch call, then evaluates each position.

    Returns {"stopped": N, "trailing_stopped": N}.
    """
    settings = get_settings()
    counts = {"stopped": 0, "trailing_stopped": 0}

    if not settings.stop_loss_enabled and not settings.trailing_stop_enabled:
        return counts

    with get_session() as session:
        open_trades = list(
            session.exec(
                select(PaperTrade).where(PaperTrade.status == "open")
            ).all()
        )

    if not open_trades:
        return counts

    # Batch-fetch all prices in one call
    prices = _fetch_current_prices(open_trades)

    for trade in open_trades:
        # Skip positions opened in the last 2 minutes (avoid entry volatility)
        age = _position_age_minutes(trade)
        if age < 2:
            continue

        current_price = prices.get(trade.coin)
        if current_price is None:
            logger.debug(f"No price data for {trade.coin} — skipping stop-loss check")
            continue

        # Compute unrealized PnL %
        pnl_pct = _unrealized_pnl_pct(trade, current_price)

        # Update trailing stop high
        if settings.trailing_stop_enabled:
            _update_trailing_high(trade, current_price, pnl_pct)

        # Check hard stop
        if settings.stop_loss_enabled and pnl_pct <= settings.stop_loss_pct:
            _execute_stop(trade, current_price, pnl_pct, reason="stop_loss")
            counts["stopped"] += 1
            continue

        # Check trailing stop
        if (
            settings.trailing_stop_enabled
            and trade.trailing_stop_high is not None
            and pnl_pct > settings.trailing_stop_activation_pct  # activated
        ):
            trailing_pnl = (current_price - trade.trailing_stop_high) / trade.trailing_stop_high
            if trailing_pnl <= settings.trailing_stop_pct:
                _execute_stop(
                    trade, current_price, pnl_pct,
                    reason=f"trailing_stop (from high ${trade.trailing_stop_high:.4f})",
                )
                counts["trailing_stopped"] += 1

    if any(v > 0 for v in counts.values()):
        logger.info(f"Stop-loss results: {counts}")

    return counts


def _fetch_current_prices(trades: list[PaperTrade]) -> dict[str, float]:
    """Fetch current Bitget spot prices for all unique coins."""
    coins = list({t.coin for t in trades})
    prices: dict[str, float] = {}

    client = BitgetClient()
    for coin in coins:
        try:
            symbol = hl_coin_to_bitget_symbol(coin)
            ticker = client.get_ticker(symbol)
            if ticker and "last" in ticker:
                prices[coin] = float(ticker["last"])
        except Exception as e:
            logger.warning(f"Failed to fetch price for {coin}: {e}")

    return prices


def _unrealized_pnl_pct(trade: PaperTrade, current_price: float) -> float:
    """Compute unrealized PnL as percentage of stake."""
    if trade.entry_price <= 0 or trade.stake_amount <= 0:
        return 0.0
    gross_pnl = (current_price - trade.entry_price) * trade.quantity
    return gross_pnl / trade.stake_amount


def _position_age_minutes(trade: PaperTrade) -> float:
    """How many minutes since the position was opened."""
    now = datetime.now(UTC).replace(tzinfo=None)
    return (now - trade.opened_at).total_seconds() / 60


def _update_trailing_high(
    trade: PaperTrade, current_price: float, pnl_pct: float
) -> None:
    """Update the trailing stop high-water mark if position is profitable."""
    if pnl_pct <= 0:
        return

    current_high = trade.trailing_stop_high or trade.entry_price
    if current_price > current_high:
        with get_session() as session:
            db_trade = session.exec(
                select(PaperTrade).where(PaperTrade.id == trade.id)
            ).first()
            if db_trade:
                db_trade.trailing_stop_high = current_price
                session.add(db_trade)
                session.commit()
                trade.trailing_stop_high = current_price  # update local copy


def _execute_stop(
    trade: PaperTrade,
    current_price: float,
    pnl_pct: float,
    reason: str,
) -> None:
    """Close a position at market due to stop-loss trigger."""
    from app.paper.pnl import calculate_close_fee, calculate_gross_pnl, calculate_net_pnl

    gross_pnl = calculate_gross_pnl(trade.entry_price, current_price, trade.quantity)
    fee = calculate_close_fee(current_price, trade.quantity)
    net_pnl = calculate_net_pnl(gross_pnl, fee)

    with get_session() as session:
        db_trade = session.exec(
            select(PaperTrade).where(PaperTrade.id == trade.id)
        ).first()
        if db_trade is None or db_trade.status != "open":
            return  # Already closed

        db_trade.status = "closed"
        db_trade.exit_price = current_price
        db_trade.closed_at = datetime.now(UTC).replace(tzinfo=None)
        db_trade.pnl_amount = net_pnl
        db_trade.pnl_pct = net_pnl / trade.stake_amount if trade.stake_amount else 0.0
        db_trade.fee_estimate = (db_trade.fee_estimate or 0) + fee
        db_trade.close_reason = reason

        # Update portfolio balance
        from app.db.models import PaperPortfolio

        portfolio = session.exec(
            select(PaperPortfolio).where(PaperPortfolio.id == trade.portfolio_id)
        ).first()
        if portfolio:
            portfolio.current_balance += trade.stake_amount + net_pnl
            portfolio.realised_pnl = (portfolio.realised_pnl or 0) + net_pnl
            portfolio.total_fees = (portfolio.total_fees or 0) + fee
            portfolio.updated_at = datetime.now(UTC).replace(tzinfo=None)

        session.add(db_trade)
        if portfolio:
            session.add(portfolio)
        session.commit()

    logger.warning(
        f"🛑 STOP-LOSS: {trade.coin} {trade.side} closed at ${current_price:.4f} "
        f"(PnL {pnl_pct*100:+.2f}%, stake ${trade.stake_amount:.2f}, "
        f"net ${net_pnl:+.2f}) — {reason}"
    )
```

---

## Task 3: Wire into signal processor

**File:** `app/paper/signal_processor.py`

In `process_all_signals()`, add after the stale position check:

```python
    # Check stop-losses BEFORE processing new signals.
    # Frees capital from stopped-out positions for new signals in this tick.
    stop_counts = check_stop_losses()
    if any(v > 0 for v in stop_counts.values()):
        logger.info(f"Stop-loss results: {stop_counts}")
```

---

## Task 4: Handle migration edge case

**Edge case:** Existing open trades won't have `trailing_stop_high` set. On first tick:
- `_update_trailing_high` initializes it from `entry_price` if null
- No special migration needed — just the ALTER TABLE

---

## Task 5: Write tests

**File:** `tests/test_stop_loss.py`

Core test cases:
1. `test_hard_stop_triggers_at_threshold` — PnL ≤ -15% → position closed
2. `test_hard_stop_does_not_trigger_above_threshold` — PnL = -10% → position stays open
3. `test_position_under_2_minutes_skipped` — new position → no check
4. `test_trailing_stop_updates_high` — price rises → trailing_high updated
5. `test_trailing_stop_triggers_from_peak` — price drops 10% from peak → closed
6. `test_trailing_stop_not_activated_below_activation` — PnL < 5% → trailing inactive
7. `test_no_price_data_skips` — ticker fails → position skipped (not closed blindly)
8. `test_both_stops_hard_wins` — price below hard stop AND trailing → hard stop triggers first
9. `test_disabled_does_nothing` — both off → no positions touched
10. `test_portfolio_balance_updated_on_stop` — balance + PnL updated correctly

---

## Verification Checklist

- [ ] Position PnL ≤ -15% → closed with reason "stop_loss"
- [ ] Position PnL > -15% → left open
- [ ] Position < 2 minutes old → skipped (no false stop on entry)
- [ ] Trailing high updates as price rises
- [ ] Trailing stop triggers when price drops 10% from peak
- [ ] Both stops enabled → exits at better price
- [ ] Portfolio balance correctly updated after stop
- [ ] Stale position manager still works (stops run before stale check)
- [ ] No API keys needed (Bitget ticker is public)
