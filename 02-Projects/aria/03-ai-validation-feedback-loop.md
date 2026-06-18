# AI Validation Feedback Loop — Implementation Plan

> **For Hermes:** Use subagent-driven-development skill to implement this plan task-by-task.
> **Depends on:** 01-trader-performance-dashboard

**Goal:** Close the loop on AI validation. Track which approvals won vs lost, auto-tune the confidence threshold, and produce a daily summary of the AI's performance — so it stops being a rubber stamp.

**Architecture:** When a paper trade closes, look up the AI validation cache entry that approved/rejected it. Record the outcome (win/loss + PnL) in a new `AIValidationOutcome` table. A daily job computes precision/recall per confidence band and optionally auto-adjusts `AI_VALIDATION_MIN_CONFIDENCE`.

**Tech Stack:** Python 3.11+, SQLModel, Pydantic Settings, pytest, SQLite

---

## What This Upgrade Does

### The Problem It Solves

Aria's AI validator approved 12 of 13 signals at ~0.60–0.65 confidence. It never rejected a single one. Is it actually filtering bad trades — or just approving everything?

Without outcome tracking, you can't answer:
- What % of AI-approved trades were profitable?
- Would raising the confidence threshold from 0.60 to 0.70 have saved money?
- Is the AI catching bad trades that a human would miss — or just adding latency?

### What This Upgrade Changes

Adds a feedback loop:

```
Signal → AI validates → Trade opens → Trade closes → Record outcome against verdict
                                                              ↓
                                              Daily: "AI prevented $X in losses,
                                                      missed $Y in gains"
                                                              ↓
                                              Auto-tune: raise threshold if
                                              approval win rate drops below 40%
```

A new `AIValidationOutcome` table links each closed trade to its AI verdict. The daily report gets an "AI Validator Report Card" section. Optional auto-tuning raises `AI_VALIDATION_MIN_CONFIDENCE` if the AI is too permissive.

### Configurability

| Setting | Default | What It Does |
|---------|---------|--------------|
| `AI_FEEDBACK_ENABLED` | `true` | Track outcomes and show in daily report |
| `AI_AUTO_TUNE_ENABLED` | `false` | Auto-adjust min_confidence (off by default — start supervised) |
| `AI_AUTO_TUNE_TARGET_WIN_RATE` | `0.45` | Target: 45% of approved trades should be profitable |
| `AI_AUTO_TUNE_STEP` | `0.05` | How much to raise/lower threshold per adjustment |

### Scope

- Applies to **closed paper trades with AI validation records**
- Links trade outcome back to `AIValidationCache` entry
- Daily report section shows: total validated, approved, rejected, win rates by confidence band
- Auto-tuning is **off by default** — you review the report first, then enable

---

## Files to Touch

| File | Action | Purpose |
|------|--------|---------|
| `app/db/models.py` | Modify | Add `AIValidationOutcome` model |
| `app/core/config.py` | Modify | Add AI feedback config settings |
| `.env.example` | Modify | Add `AI_FEEDBACK_*` env vars |
| `app/ai/feedback.py` | **Create** | Outcome recording + report generation + auto-tuning |
| `app/paper/execution.py` | Modify | Call `record_ai_outcome()` on trade close |
| `app/reports/daily.py` | Modify | Add AI validator report card section |
| `tests/test_ai_feedback.py` | **Create** | Tests for outcome tracking and auto-tuning |

---

## Task 1: Add DB model and config

**File:** `app/db/models.py`

After `AIValidationCache`:

```python
class AIValidationOutcome(SQLModel, table=True):
    """Links a closed trade to its AI validation verdict for feedback."""

    __tablename__ = "ai_validation_outcomes"

    id: int | None = Field(default=None, primary_key=True)
    paper_trade_id: int = Field(index=True)
    coin: str = ""
    side: str = "long"
    # The AI verdict
    ai_approved: bool = False
    ai_confidence: float = 0.0
    ai_reason: str = ""
    # The actual outcome
    trade_pnl: float = 0.0
    trade_pnl_pct: float = 0.0
    trade_profitable: bool = False  # pnl > 0
    # Timing
    validated_at: datetime | None = None  # when AI cache entry was created
    trade_closed_at: datetime = Field(default_factory=utc_now)
```

**File:** `app/core/config.py`

```python
    # AI Feedback Loop
    ai_feedback_enabled: bool = True
    ai_auto_tune_enabled: bool = False
    ai_auto_tune_target_win_rate: float = 0.45
    ai_auto_tune_step: float = 0.05
```

---

## Task 2: Build feedback engine

**File:** `app/ai/feedback.py`

```python
"""AI validation feedback — tracks outcomes, generates report cards, auto-tunes."""

from __future__ import annotations

from datetime import UTC, datetime, timedelta

from loguru import logger
from sqlmodel import func, select

from app.core.config import get_settings
from app.db.models import AIValidationCache, AIValidationOutcome
from app.db.session import get_session


def record_ai_outcome(
    paper_trade_id: int,
    coin: str,
    side: str,
    trade_pnl: float,
    trade_pnl_pct: float,
) -> None:
    """Record a trade's outcome against its AI validation verdict.

    Called on trade close. Looks up the most recent AI cache entry
    for this coin+side that predates the trade close.
    """
    settings = get_settings()
    if not settings.ai_feedback_enabled:
        return

    # Find the AI verdict that approved this trade
    with get_session() as session:
        ai_entry = session.exec(
            select(AIValidationCache)
            .where(
                AIValidationCache.coin == coin,
                AIValidationCache.side == side,
            )
            .order_by(AIValidationCache.researched_at.desc())
        ).first()

        if ai_entry is None:
            return  # No AI verdict — nothing to track

        outcome = AIValidationOutcome(
            paper_trade_id=paper_trade_id,
            coin=coin,
            side=side,
            ai_approved=ai_entry.approved,
            ai_confidence=ai_entry.confidence,
            ai_reason=ai_entry.reason,
            trade_pnl=trade_pnl,
            trade_pnl_pct=trade_pnl_pct,
            trade_profitable=trade_pnl > 0,
            validated_at=ai_entry.researched_at,
        )
        session.add(outcome)
        session.commit()


def generate_ai_report_card() -> str:
    """Generate a human-readable AI validator performance summary."""
    with get_session() as session:
        outcomes = list(
            session.exec(
                select(AIValidationOutcome).order_by(
                    AIValidationOutcome.trade_closed_at.desc()
                ).limit(100)
            ).all()
        )

    if not outcomes:
        return "No AI-validated trades closed yet."

    total = len(outcomes)
    wins = sum(1 for o in outcomes if o.trade_profitable)
    total_pnl = sum(o.trade_pnl for o in outcomes)
    approved_total = sum(1 for o in outcomes if o.ai_approved)
    approved_wins = sum(1 for o in outcomes if o.ai_approved and o.trade_profitable)
    approved_pnl = sum(o.trade_pnl for o in outcomes if o.ai_approved)

    # By confidence band
    bands = [
        ("0.90-1.00", 0.90, 1.00),
        ("0.75-0.90", 0.75, 0.90),
        ("0.60-0.75", 0.60, 0.75),
        ("0.50-0.60", 0.50, 0.60),
        ("0.00-0.50", 0.00, 0.50),
    ]

    lines = [
        "\n## 🤖 AI Validator Report Card\n",
        f"Total validated: {total} | Wins: {wins} ({wins/max(total,1)*100:.0f}%)",
        f"Total PnL on validated trades: ${total_pnl:+.2f}",
        f"Approved: {approved_total}/{total} ({approved_total/max(total,1)*100:.0f}%) — {approved_wins} wins, PnL ${approved_pnl:+.2f}",
        "",
        "By confidence band:",
        f"{'Band':<12} {'Trades':>7} {'Win%':>7} {'PnL':>10}",
        "-" * 40,
    ]

    for label, lo, hi in bands:
        in_band = [o for o in outcomes if lo <= o.ai_confidence < hi]
        if not in_band:
            continue
        bwins = sum(1 for o in in_band if o.trade_profitable)
        bpnl = sum(o.trade_pnl for o in in_band)
        lines.append(
            f"{label:<12} {len(in_band):>7} {bwins/max(len(in_band),1)*100:>6.0f}% "
            f"${bpnl:>9.2f}"
        )

    return "\n".join(lines)


def maybe_auto_tune() -> None:
    """Auto-adjust AI_VALIDATION_MIN_CONFIDENCE based on recent outcomes.

    Only runs if AI_AUTO_TUNE_ENABLED=true. Checks last 20 approved trades.
    If win rate < target, raises threshold. If win rate >> target, lowers.
    """
    settings = get_settings()
    if not settings.ai_auto_tune_enabled or not settings.ai_feedback_enabled:
        return

    with get_session() as session:
        recent = list(
            session.exec(
                select(AIValidationOutcome)
                .where(AIValidationOutcome.ai_approved == True)  # noqa: E712
                .order_by(AIValidationOutcome.trade_closed_at.desc())
                .limit(20)
            ).all()
        )

    if len(recent) < 10:
        return  # Not enough data

    wins = sum(1 for o in recent if o.trade_profitable)
    win_rate = wins / len(recent)

    current = settings.ai_validation_min_confidence
    target = settings.ai_auto_tune_target_win_rate
    step = settings.ai_auto_tune_step
    new_threshold = current

    if win_rate < target - 0.10:
        # Far below target — raise threshold significantly
        new_threshold = min(current + step * 2, 0.90)
        logger.warning(
            f"AI auto-tune: win rate {win_rate:.0%} << target {target:.0%}. "
            f"Raising min_confidence {current:.2f} → {new_threshold:.2f}"
        )
    elif win_rate < target:
        new_threshold = min(current + step, 0.90)
        logger.info(
            f"AI auto-tune: win rate {win_rate:.0%} < target {target:.0%}. "
            f"Raising min_confidence {current:.2f} → {new_threshold:.2f}"
        )
    elif win_rate > target + 0.15:
        new_threshold = max(current - step, 0.40)
        logger.info(
            f"AI auto-tune: win rate {win_rate:.0%} > target {target:.0%}. "
            f"Lowering min_confidence {current:.2f} → {new_threshold:.2f}"
        )

    if new_threshold != current:
        # Write to BotSetting so it survives restarts
        from app.db.models import BotSetting
        from app.core.time import utc_now

        with get_session() as session:
            row = session.exec(
                select(BotSetting).where(
                    BotSetting.key == "ai_validation_min_confidence"
                )
            ).first()
            if row:
                row.value = str(new_threshold)
                row.updated_at = utc_now()
            else:
                session.add(
                    BotSetting(
                        key="ai_validation_min_confidence",
                        value=str(new_threshold),
                    )
                )
            session.commit()
        logger.info(f"AI min_confidence auto-tuned: {current:.2f} → {new_threshold:.2f}")
```

---

## Task 3: Wire outcome recording into trade close

**File:** `app/paper/execution.py` (or wherever trade close logic lives)

After setting `trade.status = "closed"` and computing PnL:

```python
# Record AI validation outcome for feedback loop
if settings.ai_feedback_enabled and trade.trader:
    from app.ai.feedback import record_ai_outcome
    record_ai_outcome(
        paper_trade_id=trade.id,
        coin=trade.coin,
        side=trade.side,
        trade_pnl=trade.pnl_amount or 0.0,
        trade_pnl_pct=trade.pnl_pct or 0.0,
    )
```

---

## Task 4: Add AI report card to daily report

**File:** `app/reports/daily.py`

```python
# At the end of generate_daily_report():
from app.ai.feedback import generate_ai_report_card, maybe_auto_tune

report += generate_ai_report_card()

# Auto-tune after report generation (once per day)
maybe_auto_tune()
```

---

## Task 5: Write tests

**File:** `tests/test_ai_feedback.py`

Core test cases:
1. `test_record_outcome_links_to_cache` — outcome references correct AI verdict
2. `test_no_cache_entry_skips` — trade without AI verdict → no outcome recorded
3. `test_report_card_formatting` — generates correct markdown
4. `test_auto_tune_raises_on_low_win_rate` — 30% win rate → threshold raised
5. `test_auto_tune_lowers_on_high_win_rate` — 70% win rate → threshold lowered
6. `test_auto_tune_disabled_does_nothing` — when off, no changes
7. `test_auto_tune_insufficient_data` — < 10 trades → no adjustment

---

## Verification Checklist

- [ ] Closing a trade with AI verdict → outcome row created
- [ ] Closing a trade without AI verdict → nothing created (no crash)
- [ ] Daily report shows AI report card with confidence bands
- [ ] Auto-tune disabled → no threshold changes
- [ ] Auto-tune enabled + low win rate → threshold raises, stored in BotSetting
- [ ] `BotSetting` override survives restarts (checked at startup)
