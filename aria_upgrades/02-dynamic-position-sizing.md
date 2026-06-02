# Dynamic Position Sizing — Implementation Plan

> **For Hermes:** Use subagent-driven-development skill to implement this plan task-by-task.
> **Depends on:** 01-trader-performance-dashboard

**Goal:** Stop allocating the same stake to every trade. Scale position size based on trader quality and AI confidence — cut exposure to losing traders, boost it for winners.

**Architecture:** A `calculate_dynamic_stake()` function in the risk module that takes `base_stake × trader_multiplier × ai_confidence_multiplier`. Multipliers are derived from `TraderStats` (7-day win rate) and `AIValidationCache` (confidence score). Final stake is capped by `max_stake_pct`.

**Tech Stack:** Python 3.11+, SQLModel, Pydantic Settings, pytest, SQLite

---

## What This Upgrade Does

### The Problem It Solves

Every trade gets `default_stake_pct × leverage` — exactly the same, regardless of signal quality. A trade from a trader with 80% win rate and 0.95 AI confidence gets the same allocation as one from a 30% win-rate trader with 0.55 confidence.

From your actual data: the XRP trade (-$4.94%, $199.87 stake) and the ORDI trade (+$3.81%, $149.07 stake) got nearly the same allocation. If you'd known the XRP trader was weak, you'd have cut that stake in half — saving ~$5.

### What This Upgrade Changes

Replace fixed `default_stake_pct` with a dynamic formula:

```
final_stake = base_stake × trader_mult × ai_mult
```

| Component | Range | Derived From |
|-----------|-------|--------------|
| `base_stake` | 2.5% (configurable) | `default_stake_pct` |
| `trader_mult` | 0.5× – 1.5× | Trader's 7-day win rate |
| `ai_mult` | 0.7× – 1.2× | AI validation confidence |

**Trader multiplier mapping:**
- Win rate < 30% → 0.5× (cut allocation in half)
- Win rate 30–40% → 0.75×
- Win rate 40–55% → 1.0× (baseline)
- Win rate 55–70% → 1.25×
- Win rate > 70% → 1.5× (max boost)
- Trader with < 5 trades → 1.0× (not enough data)

**AI confidence multiplier mapping:**
- Confidence < 0.50 → 0.7×
- Confidence 0.50–0.60 → 0.85×
- Confidence 0.60–0.75 → 1.0×
- Confidence > 0.75 → 1.2×

Final stake is capped at `max_stake_pct` (default 5%) and floored at `min_dynamic_stake_pct` (new config, default 1%).

### Configurability

| Setting | Default | What It Does |
|---------|---------|--------------|
| `DYNAMIC_STAKING_ENABLED` | `true` | Master switch. False = use fixed `default_stake_pct` |
| `MIN_DYNAMIC_STAKE_PCT` | `0.01` | Floor: never go below 1% of equity |
| `MAX_DYNAMIC_STAKE_PCT` | `0.05` | Ceiling (same as existing `max_stake_pct`) |
| `TRADER_MULT_ENABLED` | `true` | Toggle trader-based multiplier |
| `AI_CONFIDENCE_MULT_ENABLED` | `true` | Toggle AI-confidence multiplier |

### Scope

- Applies to **all new Invo open signals**
- Does NOT affect existing open positions
- Does NOT affect signal acceptance (just sizing)
- Falls back to `default_stake_pct` if trader has no stats yet
- Multipliers are recomputed at signal time (always fresh)

---

## Files to Touch

| File | Action | Purpose |
|------|--------|---------|
| `app/core/config.py` | Modify | Add dynamic staking config settings |
| `.env.example` | Modify | Add `DYNAMIC_STAKING_*` env vars |
| `app/risk/dynamic_stake.py` | **Create** | Multiplier computation + `calculate_dynamic_stake()` |
| `app/invo/processor.py` | Modify | Call dynamic stake instead of fixed |
| `app/paper/execution.py` | Modify | Accept optional override stake |
| `tests/test_dynamic_stake.py` | **Create** | Tests for multiplier logic |

---

## Task 1: Add config settings

**File:** `app/core/config.py`

Add after the existing stake settings (`max_stake_pct`):

```python
    # Dynamic Stake Sizing
    dynamic_staking_enabled: bool = True
    min_dynamic_stake_pct: float = 0.01
    max_dynamic_stake_pct: float = 0.05
    trader_mult_enabled: bool = True
    ai_confidence_mult_enabled: bool = True
```

**File:** `.env.example`

```bash
# Dynamic Stake Sizing
DYNAMIC_STAKING_ENABLED=true
MIN_DYNAMIC_STAKE_PCT=0.01
MAX_DYNAMIC_STAKE_PCT=0.05
TRADER_MULT_ENABLED=true
AI_CONFIDENCE_MULT_ENABLED=true
```

---

## Task 2: Build dynamic stake calculator

**File:** `app/risk/dynamic_stake.py`

```python
"""Dynamic position sizing — scales stake by trader quality and AI confidence."""

from __future__ import annotations

from loguru import logger
from sqlmodel import select

from app.core.config import get_settings
from app.db.models import AIValidationCache, TraderStats
from app.db.session import get_session


def calculate_dynamic_stake(
    base_stake: float,
    trader: str | None = None,
    coin: str = "",
    side: str = "long",
) -> tuple[float, str]:
    """Calculate a dynamically-scaled stake amount.

    Returns (stake_amount, debug_reason).
    """
    settings = get_settings()

    if not settings.dynamic_staking_enabled:
        return base_stake, "dynamic staking disabled"

    equity_pct = base_stake  # base_stake comes in as % of equity
    multiplier = 1.0
    reasons: list[str] = []

    # ── Trader multiplier ──
    if settings.trader_mult_enabled and trader:
        t_mult = _get_trader_multiplier(trader)
        multiplier *= t_mult
        if t_mult != 1.0:
            reasons.append(f"trader={t_mult:.2f}x")

    # ── AI confidence multiplier ──
    if settings.ai_confidence_mult_enabled and coin:
        ai_mult = _get_ai_multiplier(coin, side)
        multiplier *= ai_mult
        if ai_mult != 1.0:
            reasons.append(f"ai={ai_mult:.2f}x")

    # Apply multiplier to base stake
    adjusted_pct = equity_pct * multiplier

    # Clamp
    final_pct = max(
        settings.min_dynamic_stake_pct,
        min(adjusted_pct, settings.max_dynamic_stake_pct),
    )

    if multiplier != 1.0:
        logger.info(
            f"Dynamic stake: {equity_pct*100:.1f}% → {final_pct*100:.1f}% "
            f"(mult={multiplier:.2f}, {' '.join(reasons)})"
        )

    debug_reason = f"dyn={final_pct*100:.1f}% mult={multiplier:.2f}"
    if reasons:
        debug_reason += f" ({', '.join(reasons)})"

    return final_pct, debug_reason


def _get_trader_multiplier(trader: str) -> float:
    """Get stake multiplier based on trader's 7-day win rate."""
    with get_session() as session:
        stats = session.exec(
            select(TraderStats).where(TraderStats.trader == trader)
        ).first()

    if stats is None or stats.trades_7d < 5:
        return 1.0  # Not enough data

    win_rate = stats.wins_7d / max(stats.trades_7d, 1)

    if win_rate < 0.30:
        return 0.5
    elif win_rate < 0.40:
        return 0.75
    elif win_rate < 0.55:
        return 1.0
    elif win_rate < 0.70:
        return 1.25
    else:
        return 1.5


def _get_ai_multiplier(coin: str, side: str) -> float:
    """Get stake multiplier based on AI validation confidence."""
    with get_session() as session:
        cached = session.exec(
            select(AIValidationCache)
            .where(
                AIValidationCache.coin == coin,
                AIValidationCache.side == side,
            )
            .order_by(AIValidationCache.researched_at.desc())
        ).first()

    if cached is None:
        return 1.0  # No AI data — neutral

    conf = cached.confidence

    if conf < 0.50:
        return 0.7
    elif conf < 0.60:
        return 0.85
    elif conf < 0.75:
        return 1.0
    else:
        return 1.2
```

---

## Task 3: Wire dynamic stake into Invo processor

**File:** `app/invo/processor.py`

In `_process_open()`, replace the fixed stake calculation with the dynamic one.

**Before (simplified):**
```python
stake_pct = settings.default_stake_pct
stake_amount = equity * stake_pct * copied_lev
```

**After:**
```python
from app.risk.dynamic_stake import calculate_dynamic_stake

stake_pct, dyn_reason = calculate_dynamic_stake(
    base_stake=settings.default_stake_pct,
    trader=payload.trader,
    coin=payload.symbol,
    side=payload.side,
)
stake_amount = equity * stake_pct * copied_lev
```

Pass `dyn_reason` through to the trade notes or log.

---

## Task 4: Write tests

**File:** `tests/test_dynamic_stake.py`

```python
class TestDynamicStake:
    def test_disabled_returns_base_stake(self):
        """When disabled, base stake is returned unchanged."""
        ...

    def test_trader_high_win_rate_boosts(self):
        """Trader with 80% win rate gets 1.5x multiplier."""
        ...

    def test_trader_low_win_rate_cuts(self):
        """Trader with 20% win rate gets 0.5x multiplier."""
        ...

    def test_trader_insufficient_data_neutral(self):
        """Trader with < 5 trades gets 1.0x."""
        ...

    def test_ai_high_confidence_boosts(self):
        """AI confidence 0.90 → 1.2x multiplier."""
        ...

    def test_ai_low_confidence_cuts(self):
        """AI confidence 0.40 → 0.7x multiplier."""
        ...

    def test_combined_multipliers(self):
        """Trader 1.5x + AI 1.2x = 1.8x combined, capped at max_stake_pct."""
        ...

    def test_floor_enforced(self):
        """Even worst multipliers don't go below min_dynamic_stake_pct."""
        ...
```

---

## Verification Checklist

- [ ] `DYNAMIC_STAKING_ENABLED=false` → uses fixed `default_stake_pct` (backward compat)
- [ ] Trader with 80% win rate → 1.5× stake
- [ ] Trader with 20% win rate → 0.5× stake
- [ ] Unknown trader → 1.0× (no penalty)
- [ ] AI confidence 0.90 → 1.2× stake
- [ ] Stake never exceeds `max_stake_pct` × equity
- [ ] Stake never goes below `min_dynamic_stake_pct` × equity
- [ ] Logs show multiplier reason for debugging
