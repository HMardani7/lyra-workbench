# Trader Performance Dashboard — Implementation Plan

> **For Hermes:** Use subagent-driven-development skill to implement this plan task-by-task.

**Goal:** Give Aria trader-level performance tracking. Track every Invo trader's trade outcomes (win rate, PnL, hold time, consistency) so the bot can later allocate capital based on merit — not equally to everyone.

**Architecture:** Add a `TraderStats` SQLModel table updated on every trade close. Add a `trader` column to `PaperTrade` for attribution. Build a `TraderTracker` service that recalculates stats (rolling 7-day/30-day windows). Expose trader rankings in the daily report.

**Tech Stack:** Python 3.11+, SQLModel, Pydantic Settings, pytest, SQLite

---

## What This Upgrade Does

### The Problem It Solves

Aria currently copies every allowed Invo trader equally. A trader with an 80% win rate and +$50 lifetime PnL gets the same 2.5% stake as one with a 30% win rate bleeding -$30. There's no feedback loop — the bot can't learn who's good and who's bad.

**Real example from current data:**
- `cobieg`: opened UNI, ARB, and others — is this trader profitable? Nobody knows.
- The XRP trade lost -$4.94% — which trader opened it? Was this a pattern or an outlier?

Without per-trader stats, every allocation decision is blind.

### What This Upgrade Changes

Adds a `TraderStats` table and tracking layer that computes per-trader metrics:

| Metric | What It Tells You |
|--------|-------------------|
| Total trades, open trades | Activity level |
| Wins / losses / win rate | Raw performance |
| Total PnL, avg PnL per trade | Profitability |
| Best trade, worst trade | Outlier detection |
| Avg hold time | Trading style (scalper vs swing) |
| 7-day rolling PnL / win rate | Recent form (what matters most) |
| 30-day rolling PnL / win rate | Long-term consistency |

Stats are recalculated on every trade close (not on a timer — always fresh).

### Scope

- Applies to **Invo signals only** (Hyperliquid wallet tracking is being removed in upgrade #5)
- `PaperTrade` gets a nullable `trader` column (populated from Invo signal's `trader` field)
- `TraderStats` table is small — one row per trader, recomputed on close
- Daily report includes a trader leaderboard section

---

## Files to Touch

| File | Action | Purpose |
|------|--------|---------|
| `app/db/models.py` | Modify | Add `trader` column to `PaperTrade`, add `TraderStats` model |
| `app/risk/trader_stats.py` | **Create** | Stats computation engine |
| `app/invo/processor.py` | Modify | Pass `trader` through to paper trade creation |
| `app/paper/execution.py` | Modify | Accept `trader` param, save to `PaperTrade` |
| `app/reports/daily.py` | Modify | Add trader leaderboard section |
| `app/jobs/tasks.py` | Modify | Add weekly trader summary task (optional) |
| `tests/test_trader_stats.py` | **Create** | Tests for stats computation |

---

## Task 1: Add DB models

**Objective:** Add `trader` column to `PaperTrade` and create `TraderStats` table.

**Files:** `app/db/models.py`

**Step 1: Add trader column to PaperTrade**

After the `notes` field in `PaperTrade`:

```python
    # Source attribution (upgrade #4)
    trader: str | None = Field(default=None, index=True)
```

**Step 2: Create TraderStats model**

After the `PaperTrade` class:

```python
class TraderStats(SQLModel, table=True):
    """Per-trader performance statistics. Recalculated on every trade close."""

    __tablename__ = "trader_stats"

    id: int | None = Field(default=None, primary_key=True)
    trader: str = Field(index=True, unique=True)
    # Lifetime
    total_trades: int = 0
    wins: int = 0
    losses: int = 0
    total_pnl: float = 0.0
    avg_pnl: float = 0.0
    best_trade_pnl: float = 0.0
    worst_trade_pnl: float = 0.0
    avg_hold_minutes: float = 0.0
    # Rolling windows
    trades_7d: int = 0
    wins_7d: int = 0
    pnl_7d: float = 0.0
    trades_30d: int = 0
    wins_30d: int = 0
    pnl_30d: float = 0.0
    # Metadata
    first_trade_at: datetime | None = None
    last_trade_at: datetime | None = None
    updated_at: datetime = Field(default_factory=utc_now)
```

**Step 3: Add migration for existing PaperTrade rows**

The `_run_migrations()` function in `init_db.py` already handles ALTER TABLE. Add:

```python
    # Migration: add trader column to paper_trades
    if "paper_trades" in inspector.get_table_names():
        existing = {c["name"] for c in inspector.get_columns("paper_trades")}
        if "trader" not in existing:
            with engine.connect() as conn:
                conn.execute(text("ALTER TABLE paper_trades ADD COLUMN trader VARCHAR"))
                conn.commit()
            logger.info("Migration: added column paper_trades.trader")
```

**Verify:**

```bash
cd /root/hyperliquid-copy-bot
uv run python -c "
from app.db.init_db import init_database
from app.db.models import PaperTrade, TraderStats
init_database()
print('trader column:', hasattr(PaperTrade, 'trader'))
print('TraderStats table:', TraderStats.__tablename__)
"
```

---

## Task 2: Build TraderStats computation engine

**Objective:** Create `app/risk/trader_stats.py` that recalculates stats for a trader.

**File:** `app/risk/trader_stats.py`

```python
"""Trader performance tracking — computes per-trader stats on trade close."""

from __future__ import annotations

from datetime import UTC, datetime, timedelta

from loguru import logger
from sqlmodel import func, select

from app.db.models import PaperTrade, TraderStats
from app.db.session import get_session


def recompute_trader_stats(trader: str) -> TraderStats | None:
    """Recompute all stats for a trader after a trade closes.

    Called from the trade close path. Computes lifetime + rolling 7d/30d.
    """
    now = datetime.now(UTC).replace(tzinfo=None)
    cutoff_7d = now - timedelta(days=7)
    cutoff_30d = now - timedelta(days=30)

    with get_session() as session:
        # Get or create stats row
        stats = session.exec(
            select(TraderStats).where(TraderStats.trader == trader)
        ).first()
        if stats is None:
            stats = TraderStats(trader=trader)
            session.add(stats)
            session.commit()
            session.refresh(stats)

        # ── Lifetime stats ──
        lifetime = _compute_window(session, trader, cutoff=None)
        stats.total_trades = lifetime["trades"]
        stats.wins = lifetime["wins"]
        stats.losses = lifetime["losses"]
        stats.total_pnl = lifetime["total_pnl"]
        stats.avg_pnl = lifetime["avg_pnl"]
        stats.best_trade_pnl = lifetime["best_pnl"]
        stats.worst_trade_pnl = lifetime["worst_pnl"]
        stats.avg_hold_minutes = lifetime["avg_hold_min"]
        stats.first_trade_at = lifetime["first_trade_at"]
        stats.last_trade_at = lifetime["last_trade_at"]

        # ── 7-day rolling ──
        w7 = _compute_window(session, trader, cutoff=cutoff_7d)
        stats.trades_7d = w7["trades"]
        stats.wins_7d = w7["wins"]
        stats.pnl_7d = w7["total_pnl"]

        # ── 30-day rolling ──
        w30 = _compute_window(session, trader, cutoff=cutoff_30d)
        stats.trades_30d = w30["trades"]
        stats.wins_30d = w30["wins"]
        stats.pnl_30d = w30["total_pnl"]

        stats.updated_at = now

        session.add(stats)
        session.commit()
        session.refresh(stats)

        return stats


def _compute_window(session, trader: str, cutoff: datetime | None) -> dict:
    """Compute stats for a time window. cutoff=None means lifetime."""
    q = select(PaperTrade).where(
        PaperTrade.trader == trader,
        PaperTrade.status == "closed",
        PaperTrade.pnl_amount.isnot(None),  # type: ignore[union-attr]
    )
    if cutoff is not None:
        q = q.where(PaperTrade.closed_at >= cutoff)  # type: ignore[operator]

    trades = list(session.exec(q).all())

    if not trades:
        return {
            "trades": 0, "wins": 0, "losses": 0,
            "total_pnl": 0.0, "avg_pnl": 0.0,
            "best_pnl": 0.0, "worst_pnl": 0.0,
            "avg_hold_min": 0.0,
            "first_trade_at": None, "last_trade_at": None,
        }

    pnls = [t.pnl_amount for t in trades if t.pnl_amount is not None]
    hold_times = [
        ((t.closed_at - t.opened_at).total_seconds() / 60)
        for t in trades
        if t.closed_at and t.opened_at
    ]

    return {
        "trades": len(trades),
        "wins": sum(1 for p in pnls if p > 0),
        "losses": sum(1 for p in pnls if p < 0),
        "total_pnl": round(sum(pnls), 4),
        "avg_pnl": round(sum(pnls) / len(pnls), 4) if pnls else 0.0,
        "best_pnl": round(max(pnls), 4) if pnls else 0.0,
        "worst_pnl": round(min(pnls), 4) if pnls else 0.0,
        "avg_hold_min": round(sum(hold_times) / len(hold_times), 1) if hold_times else 0.0,
        "first_trade_at": min(t.closed_at for t in trades if t.closed_at),
        "last_trade_at": max(t.closed_at for t in trades if t.closed_at),
    }


def get_trader_leaderboard(top_n: int = 20) -> list[dict]:
    """Return traders ranked by 7-day PnL (best first)."""
    with get_session() as session:
        rows = session.exec(
            select(TraderStats)
            .order_by(TraderStats.pnl_7d.desc())  # type: ignore[attr-defined]
            .limit(top_n)
        ).all()

    return [
        {
            "trader": r.trader,
            "win_rate": round(r.wins / max(r.total_trades, 1) * 100, 1),
            "total_pnl": r.total_pnl,
            "pnl_7d": r.pnl_7d,
            "pnl_30d": r.pnl_30d,
            "trades_7d": r.trades_7d,
            "win_rate_7d": round(r.wins_7d / max(r.trades_7d, 1) * 100, 1),
            "avg_pnl": r.avg_pnl,
            "avg_hold_min": r.avg_hold_minutes,
        }
        for r in rows
    ]
```

---

## Task 3: Wire trader attribution through trade lifecycle

**Objective:** Pass `trader` from Invo signal → paper trade → stats recompute on close.

**Files:**
- `app/invo/processor.py`
- `app/paper/execution.py`

**Step 1: Invo processor — pass trader to _execute_open**

In `app/invo/processor.py`, find the `_execute_open` call and add the trader:

```python
# Before (around line 195-200 in the current processor):
result = _execute_open(payload, copied_lev)

# After:
result = _execute_open(payload, copied_lev, trader=payload.trader)
```

Similarly in `_execute_open`, accept the parameter and pass through to the actual trade creation.

**Step 2: Paper execution — save trader on PaperTrade**

In `app/paper/execution.py`, where `PaperTrade` is created:

```python
trade = PaperTrade(
    ...
    trader=trader,  # NEW
    notes=position_key_info,
)
```

**Step 3: On trade close — recompute trader stats**

In the close-signal handling code (in `app/paper/execution.py` or wherever `PaperTrade.status = "closed"` is set), add after the close:

```python
if trade.trader:
    from app.risk.trader_stats import recompute_trader_stats
    recompute_trader_stats(trade.trader)
```

---

## Task 4: Add trader leaderboard to daily report

**Objective:** Show trader rankings in Aria's daily report.

**File:** `app/reports/daily.py`

Add a new section after the existing PnL breakdown:

```python
def _trader_leaderboard_section() -> str:
    """Generate trader performance leaderboard for daily report."""
    from app.risk.trader_stats import get_trader_leaderboard

    leaders = get_trader_leaderboard(top_n=10)
    if not leaders:
        return ""

    lines = ["\n## 📊 Trader Leaderboard (7-day PnL)\n"]
    lines.append("```")
    lines.append(f"{'Trader':<15} {'Win%':>6} {'PnL 7d':>10} {'Trades':>8} {'Avg PnL':>10}")
    lines.append("-" * 55)
    for t in leaders:
        lines.append(
            f"{t['trader']:<15} {t['win_rate_7d']:>5.0f}% "
            f"${t['pnl_7d']:>9.2f} {t['trades_7d']:>8} "
            f"${t['avg_pnl']:>9.2f}"
        )
    lines.append("```")
    return "\n".join(lines)
```

**Verify:** Trigger a manual daily report:

```bash
uv run python -c "from app.reports.daily import generate_daily_report; print(generate_daily_report())"
```

Expected: report includes trader section (empty if no trades yet, populated after trades close).

---

## Task 5: Write tests

**Objective:** Validate stats computation.

**File:** `tests/test_trader_stats.py`

Core test cases:
1. `test_empty_trader_returns_zeros` — no trades → all zeros
2. `test_single_win_computes_correctly` — one winning trade → correct win rate, PnL
3. `test_mixed_trades` — 3 wins, 2 losses → correct win rate 60%, total PnL
4. `test_rolling_windows` — trades within 7d vs older → correct window separation
5. `test_leaderboard_ordering` — traders sorted by PnL descending
6. `test_avg_hold_time` — correct minutes calculation

---

## Verification Checklist

- [ ] `trader` column exists on `paper_trades`
- [ ] `trader_stats` table created
- [ ] New Invo open signals populate `trader` on the paper trade
- [ ] Closing a trade triggers `recompute_trader_stats()`
- [ ] Leaderboard function returns correct ordering
- [ ] Daily report includes trader section
- [ ] Existing trades without `trader` (null) don't break stats
- [ ] Zero-division handled (trader with 0 trades)
