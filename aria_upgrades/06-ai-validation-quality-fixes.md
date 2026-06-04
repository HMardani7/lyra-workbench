# AI Validation Quality Fixes — Implementation Plan

> **For Hermes:** Use subagent-driven-development skill to implement this plan task-by-task.
> **Status:** Fixes live issues discovered 2026-06-04 — AI rejecting profitable trades while approving losing ones.

**Goal:** Make AI validation a net-positive filter by injecting trader performance data into the prompt, fixing the "default to REJECT" bias, adding major-coin special handling, fixing the cache persistence bug, and enabling auto-tune.

**Architecture:** Three changes to `researcher.py` (trader stats injection, prompt tuning, BTC/ETH handling), one bugfix to cache persistence, one config change to enable auto-tune. No new tables, no new files.

**Tech Stack:** Python 3.12, SQLModel, DeepSeek API (v4-flash), Tavily Search, SQLite

---

## What This Upgrade Does

### The Problem It Solves

Since the event-loop fix was deployed at 15:05 on 2026-06-04, the AI has:

| Approved (4) | Result | Rejected (3) | Missed |
|---|---|---|---|
| vortex_legion BTC | -0.67% closed | WLD | unknown |
| kzs HYPE | -0.73% closed | **kzs BTC** | **+27%** in 19 min |
| blocked FET | -2.5% open | kzs PENGU | pending |
| blocked ZEC | -8.7% open | | |

**4/4 approved trades are underwater. The only clear winner was rejected.** The AI is a net-negative filter right now.

**Root causes:**

1. **No trader data in the prompt.** The model judges purely on coin/news. It doesn't know kzs has a 50% win rate. It treats every trader identically.
2. **"Default to REJECT when uncertain."** The system prompt explicitly instructs the model to default to rejection. Combined with a fast model doing shallow research, this creates false negatives on every ambiguous signal.
3. **BTC doesn't fit the framework.** The prompt asks for coin-specific catalysts and narratives. BTC rarely has a discrete "catalyst" you can search for — so the LLM finds nothing dramatic and defaults to REJECT.
4. **Cache isn't persisting.** `store_cache` logs "STORED" but the `ai_validation_cache` table is empty. Every signal for the same coin re-triggers full research even within the 15-min TTL.
5. **Auto-tune exists but is disabled.** The feedback loop (`record_outcome`, `run_auto_tune`) is fully coded and wired in, but `ai_auto_tune_enabled` is `False`.

### What This Upgrade Changes

- **Trader stats injected into every prompt** — the LLM sees win rate, total trades, avg PnL before making a decision
- **"Default to REJECT" → "Default to low-confidence APPROVE"** — shifts the model from gatekeeper to advisor; the confidence threshold still makes the final call
- **BTC/ETH get macro-focused prompts** — asks about ETF flows, market sentiment, BTC dominance instead of coin-specific catalysts
- **Cache actually persists** — bugfix so subsequent signals for the same coin within TTL use the cached verdict
- **Auto-tune enabled** — confidence threshold adjusts automatically based on 7-day win rate

### Risk It Prevents

- **Missed profits:** AI rejecting good trades from known-profitable traders because it can't see their track record
- **Redundant API costs:** Every BTC signal triggering a fresh DeepSeek + Tavily call because cache is broken
- **Stale threshold:** Confidence threshold staying at 0.60 forever regardless of performance

### Configurability

| Setting | Default | What It Does |
|---------|---------|--------------|
| `ai_auto_tune_enabled` | `true` (was `false`) | Enables automatic confidence threshold adjustment |
| `ai_validation_min_confidence` | `0.50` (was `0.60`) | Starting threshold — auto-tune will adjust from here |

### Scope

- Applies to: Invo webhook pipeline (primary) and HL signal pipeline (secondary)
- **Does** inject trader stats into the LLM prompt
- **Does** tune the system prompt
- **Does** fix the cache persistence
- **Does** enable auto-tune
- **Does not** change the model (stays `deepseek-v4-flash` for now)
- **Does not** add new database tables or services
- **Does not** change the two-turn research architecture

---

## Task 1: Fix Cache Persistence

**Objective:** Debug and fix why `store_cache()` doesn't actually persist to the `ai_validation_cache` table despite logging "STORED."

**Files:**
- Investigate: `app/ai/researcher.py:417-432` (`store_cache` function)
- Investigate: `app/db/session.py` (`get_session` / `get_engine`)
- Modify: `app/ai/researcher.py`

**Step 1: Reproduce the bug**

```bash
cd /workspace/repos/hyperliquid-copy-bot
# Run a test insert through the exact same code path
uv run python -c "
from app.ai.researcher import store_cache, ValidationResult
from dataclasses import replace

result = ValidationResult(
    approved=True, confidence=0.85, 
    reason='test cache fix', key_factors=['test']
)
store_cache('TESTCOIN', 'long', result)
print('Stored via store_cache')

from app.db.session import get_session
from app.db.models import AIValidationCache
from sqlmodel import select
with get_session() as s:
    rows = s.exec(select(AIValidationCache).where(AIValidationCache.coin == 'TESTCOIN')).all()
    print(f'Found: {len(rows)}')
    for r in rows:
        print(f'  {r.coin} {r.side} approved={r.approved}')
"
```

Expected: "Found: 1" — if it shows 0, the bug is confirmed.

**Step 2: Identify the root cause**

Check if the issue is:
- Session not committing (check `store_cache` — it does call `session.commit()`)
- Wrong database file (CWD issue)
- Exception swallowed somewhere in the async→sync thread boundary

Add explicit logging inside `store_cache` to confirm the session path.

**Step 3: Apply the fix**

Most likely fix: the `get_session()` in the async-research-called-via-thread context may use a different engine. Verify by adding `logger.info(f"DB path: {get_engine().url}")` in `store_cache`.

If the engine is correct but data still doesn't persist, wrap in a retry:

```python
def store_cache(coin: str, side: str, result: ValidationResult) -> None:
    with get_session() as session:
        cache = AIValidationCache(
            coin=coin, side=side,
            approved=result.approved, confidence=result.confidence,
            reason=result.reason,
            key_factors=json.dumps(result.key_factors),
        )
        session.add(cache)
        try:
            session.commit()
        except Exception as e:
            session.rollback()
            logger.error(f"Cache commit failed: {e}")
            raise
    # Verify it stuck
    with get_session() as s:
        count = s.exec(
            select(AIValidationCache).where(AIValidationCache.coin == coin, AIValidationCache.side == side)
        ).all()
    logger.info(f"AI cache STORED for {coin} {side}: approved={result.approved} (verified: {len(count)} rows)")
```

**Step 4: Verify**

```bash
# After restart, trigger a signal and check the DB
sqlite3 data/hyperliquid_copy_bot.db "SELECT coin, side, approved, confidence, reason FROM ai_validation_cache ORDER BY researched_at DESC LIMIT 5;"
```

Expected: recent entries visible.

**Commit:**

```bash
git add app/ai/researcher.py
git commit -m "fix: cache persistence in store_cache with verification step"
```

---

## Task 2: Inject Trader Stats into the Prompt

**Objective:** Before calling the LLM, look up the trader's stats and append them to the user prompt so the AI knows who it's evaluating.

**Files:**
- Modify: `app/ai/researcher.py` — `_call_llm_with_search` function and `USER_PROMPT_TEMPLATE`
- Read: `app/risk/trader_stats.py` — `TraderStats` model

**Step 1: Add trader stats fetching**

In `_call_llm_with_search`, before building the prompt, add:

```python
from app.db.models import TraderStats

trader_context = ""
if trader and trader != "unknown":
    with get_session() as session:
        stats = session.exec(
            select(TraderStats).where(TraderStats.trader == trader)
        ).first()
    if stats:
        trader_context = (
            f"\n\nTRADER PROFILE — {trader}:\n"
            f"- Total trades copied: {stats.total_trades}\n"
            f"- Win rate: {stats.win_rate:.0%} ({stats.wins}W / {stats.losses}L)\n"
            f"- Average PnL per trade: {stats.average_pnl_per_trade:+.1%}\n"
            f"- Total PnL: ${stats.total_pnl:+.2f}\n"
            f"- 7-day stats: {stats.trades_7d} trades, {stats.win_rate_7d:.0%} WR, {stats.pnl_7d:+.2f} PnL\n"
        )
    else:
        trader_context = f"\n\nTRADER PROFILE — {trader}: No trade history yet. Treat as unknown."
```

**Step 2: Update USER_PROMPT_TEMPLATE**

Add `{trader_context}` to the template:

```python
USER_PROMPT_TEMPLATE = """A tracked trader ({trader}) just opened a {side} position on {coin}
at approximately ${price:.2f}.
{trader_context}

Research this trade entry thoroughly:
1. Search for "{coin} price today" and "{coin} news 2026" — what's the recent price action?
2. Is {coin} pumping or dumping right now? Check 24h change.
3. Search for any specific catalysts: "{coin} catalyst", "{coin} upgrade", "{coin} partnership"
4. Search for red flags: "{coin} hack", "{coin} delist", "{coin} SEC"
5. Check broader market: search "crypto market today June 2026"

CRITICAL: The trader's profile above is REAL performance data from the copy-trading bot.
If the trader has a strong win rate (>50%) and positive average PnL, give their signals more weight.
If the trader has a poor track record, be more skeptical.
If the trader is new/unknown, evaluate purely on the coin and market conditions.

Your task: decide if this specific entry, at this specific price, by this specific trader, is worth copying.

Return ONLY your JSON verdict. No other text."""
```

Remove the old line 6 ("Consider the trader...") since the stats replace it.

**Step 3: Wire it in**

Update the `_call_llm_with_search` call to pass `trader_context`:

```python
user_prompt = USER_PROMPT_TEMPLATE.format(
    coin=coin, side=side, price=price, 
    trader=trader or "unknown",
    trader_context=trader_context
)
```

**Step 4: Verify**

```bash
cd /workspace/repos/hyperliquid-copy-bot
uv run python -c "
from app.ai.researcher import USER_PROMPT_TEMPLATE
# Verify template compiles with all fields
prompt = USER_PROMPT_TEMPLATE.format(coin='BTC', side='long', price=63000.0, trader='kzs', trader_context='TEST STATS')
assert 'kzs' in prompt
assert 'TRADER PROFILE' not in prompt  # 'TEST STATS' instead
assert 'BTC' in prompt
print('Template OK')
print(prompt[:300])
"
```

**Commit:**

```bash
git add app/ai/researcher.py
git commit -m "feat: inject trader performance stats into AI validation prompt"
```

---

## Task 3: Tune the System Prompt

**Objective:** Fix the "default to REJECT" bias and add BTC/ETH macro-handling instructions.

**Files:**
- Modify: `app/ai/researcher.py` — `SYSTEM_PROMPT` constant

**Step 1: Replace the SYSTEM_PROMPT**

Replace the entire `SYSTEM_PROMPT` string with:

```python
SYSTEM_PROMPT = """You are a crypto trade entry quality analyst for a copy-trading bot.

A tracked trader just opened a position. Your job is to evaluate whether this is a GOOD ENTRY
or a DANGEROUS ENTRY — not just whether anything is on fire.

You have access to web search. Use it aggressively to find:
1. What is this coin? What sector/narrative does it belong to?
2. What is the recent price action? Search for the current price and 24h change.
3. Is this entry at a local top (chasing a pump) or a dip (buying the pullback)?
4. Any catalysts supporting this direction? (upgrades, listings, partnerships, ETF flows)
5. Any red flags? (hacks, exploits, regulatory actions, network outages, delisting risk)
6. Market context: how is the broader crypto market doing right now?

CRITICAL RULES — follow these strictly:
- If the coin has pumped 10%+ in the last 24 hours and you cannot find a fresh catalyst justifying further upside, REJECT. Chasing pumps without new catalysts is how copy-traders lose money.
- If the trader is entering after a 3%+ intraday drop with no negative news, that's a dip buy — APPROVE with confidence 0.65+.
- If you can find a specific, credible catalyst (mainnet upgrade this week, major exchange listing announced, ETF approval news), APPROVE with confidence 0.75+.
- If the coin has no clear narrative and no catalyst, cap confidence at 0.55 — you're gambling on technicals alone.
- If the trader or coin has any association with hacks, rug pulls, or regulatory enforcement, REJECT immediately.
- If the broader market (BTC/ETH) is down 3%+ in the last 4 hours, downgrade your confidence by 0.15 — counter-trend entries are risky.

SPECIAL RULES FOR BTC AND ETH:
- BTC and ETH are macro assets — do NOT search for coin-specific "catalysts" or "narratives."
- Instead, evaluate macro context: ETF flows, institutional adoption news, Fed policy, 
  market sentiment indicators (Fear & Greed), and BTC dominance.
- For BTC/ETH, a strong trader with positive track record entering at a moderate price 
  level IS a valid signal even without a discrete catalyst. APPROVE with 0.55+ confidence 
  if the trader is proven and market conditions are neutral or positive.
- Only REJECT BTC/ETH if: clear macro headwinds (rate hike, regulatory crackdown), 
  the trader has a poor track record, OR the coin just pumped 10%+ with no catalyst.

TRADER WEIGHTING:
- The prompt includes REAL trader performance data from this bot's trade history.
- A trader with >50% win rate and positive PnL: give their signals significant weight. 
  A good trader going long BTC in neutral conditions should almost always be approved.
- A trader with <40% win rate or negative PnL: be skeptical. They need strong coin-level 
  evidence to get approved.
- Unknown traders: evaluate purely on coin and market conditions.

WHEN UNCERTAIN:
- Previously we said "Default to REJECT." This was too aggressive and caused false negatives.
- Now: if you are genuinely uncertain (no red flags, no strong catalysts, mixed market), 
  APPROVE with confidence 0.50-0.55. Let the confidence threshold make the final decision.
- Only REJECT when you have a SPECIFIC reason (pump chase, red flag, terrible trader, 
  macro headwinds). "I'm not sure" is not a rejection reason.

Return your verdict as JSON:
{
  "approved": true/false,
  "confidence": 0.0-1.0,
  "reason": "Specific explanation of your research findings and why you made this decision. Mention the trader's stats if they influenced your decision.",
  "key_factors": ["concrete factor 1", "concrete factor 2"]
}

Confidence scale:
- 0.80-1.00: Strong catalyst + good entry timing + strong trader. High conviction.
- 0.65-0.79: Moderate evidence supporting the trade. Decent entry or good trader.
- 0.50-0.64: Neutral or mixed signals. Approve if nothing negative found + trader is decent.
- 0.35-0.49: Concerning signals or weak trader. Only approve if compelling catalyst exists.
- 0.00-0.34: Clear danger, terrible entry, or known-bad trader. Always reject.

Remember: your rejections save real money. Your approvals put real money at risk.
But false rejections also cost money — a rejected trade that goes +27% is just as bad 
as an approved trade that goes -5%. Calibrate carefully."""
```

**Step 2: Verify the template compiles**

```bash
uv run python -c "from app.ai.researcher import SYSTEM_PROMPT; print(f'System prompt: {len(SYSTEM_PROMPT)} chars'); assert 'WHEN UNCERTAIN' in SYSTEM_PROMPT; assert 'BTC AND ETH' in SYSTEM_PROMPT; assert 'TRADER WEIGHTING' in SYSTEM_PROMPT; print('OK')"
```

**Commit:**

```bash
git add app/ai/researcher.py
git commit -m "fix: tune AI system prompt — reduce REJECT bias, add BTC/ETH macro handling, add trader weighting"
```

---

## Task 4: Enable Auto-Tune

**Objective:** Enable the existing `run_auto_tune()` feature and lower the starting threshold to 0.50 to let more trades through while the AI learns.

**Files:**
- Modify: `app/core/config.py`

**Step 1: Update config defaults**

In `app/core/config.py`, change:

```python
# Before
ai_validation_min_confidence: float = 0.6
ai_auto_tune_enabled: bool = False

# After  
ai_validation_min_confidence: float = 0.50
ai_auto_tune_enabled: bool = True
```

**Step 2: Update .env.example**

```bash
# In .env.example:
AI_VALIDATION_MIN_CONFIDENCE=0.50  # Starting threshold (auto-tune adjusts from here)
AI_AUTO_TUNE_ENABLED=true
AI_AUTO_TUNE_TARGET_WIN_RATE=0.50  # Target 50% win rate for approved trades
AI_AUTO_TUNE_STEP=0.05              # Adjust by 5% each time
```

**Step 3: Verify**

```bash
uv run python -c "
from app.core.config import get_settings
s = get_settings()
assert s.ai_validation_min_confidence == 0.50
assert s.ai_auto_tune_enabled == True
assert s.ai_auto_tune_target_win_rate == 0.50
print('Config OK')
"
```

**Commit:**

```bash
git add app/core/config.py .env.example
git commit -m "config: enable AI auto-tune, lower starting confidence to 0.50"
```

---

## Task 5: Rebuild and Deploy

**Objective:** Rebuild the Docker image and restart the container with all fixes.

**Step 1: Rebuild**

```bash
cd /workspace/repos/hyperliquid-copy-bot
docker build -t hyperliquid-copy-bot .
```

**Step 2: Restart**

```bash
docker stop hyperliquid-copy-bot
docker rm hyperliquid-copy-bot
docker run -d --name hyperliquid-copy-bot \
  --restart unless-stopped \
  -v /root/hyperliquid-copy-bot/data:/app/data \
  -v /root/hyperliquid-copy-bot/.env:/app/.env \
  --network host \
  hyperliquid-copy-bot
```

**Step 3: Verify health**

```bash
sleep 5
docker logs hyperliquid-copy-bot --tail 20
# Expected: "Daemon starting in PAPER mode", "Invo webhook server started"
```

**Step 4: Monitor first signal**

Wait for the next signal and check:
```bash
docker logs hyperliquid-copy-bot -f | grep -E "AI cache|trader stats|approved"
```

Expected: Research logs should now mention trader stats. Approval rate should increase from ~57% to ~70-80%.

---

## Verification Checklist

- [ ] Cache persistence: `ai_validation_cache` table has entries after a signal is processed
- [ ] Trader stats: LLM prompt includes win rate, total trades, avg PnL for known traders
- [ ] BTC/ETH handling: BTC signals don't get rejected just for lacking a "catalyst"
- [ ] "Uncertain" trades approved with 0.50-0.55 confidence instead of rejected
- [ ] Auto-tune enabled: `ai_auto_tune_enabled` is `True` in config
- [ ] Starting threshold: `ai_validation_min_confidence` is `0.50`
- [ ] Existing tests pass: `uv run pytest tests/unit/test_ai_validation.py -v`
- [ ] No new linter errors: `uv run ruff check app/ai/`
- [ ] Container starts healthy after rebuild
- [ ] First post-deploy signal processed without errors

---

## Design Decisions

| Decision | Rationale |
|----------|-----------|
| **Lower threshold to 0.50** | At 0.60, the AI rejects everything that isn't clearly good. At 0.50, it approves mild signals too — auto-tune will raise it back up if win rate drops below target. |
| **APPROVE on uncertainty instead of REJECT** | The old "default to REJECT" caused false negatives on trades that went +27%. False negatives cost as much as false positives. Letting uncertain trades through with low confidence lets auto-tune find the right threshold. |
| **BTC/ETH special handling** | These don't fit the catalyst/narrative model. Forcing the LLM to search for "BTC catalyst" produces noise and leads to default rejection. Macro-focused evaluation is more appropriate. |
| **Stick with v4-flash for now** | Upgrading to `deepseek-v4-pro` would add 2-3s latency and 2-3x cost per research call. The prompt and trader-stats fixes should improve quality enough — upgrade the model only if those don't work. |
| **Auto-tune target 0.50** | Conservative target — we want at least half of AI-approved trades to be profitable. If the AI can't beat a coin flip after these fixes, the whole feature needs rethinking. |

## Future Enhancements (not in this plan)

- **Model upgrade to v4-pro:** If v4-flash with better prompts still produces poor decisions, upgrade for deeper reasoning. Cost: ~$0.03/call vs ~$0.01/call.
- **Per-trader confidence thresholds:** kzs gets 0.45, unknown trader gets 0.60. More granular than a single global threshold.
- **Multi-turn research with follow-up searches:** Current two-turn is good but sometimes the first search misses key info. A third turn could improve depth.
- **Rejected trade tracking:** Record rejected trades' counterfactual PnL (what the trade WOULD have made) so we can measure false-negative cost.
- **Daily AI performance report in Telegram:** Auto-send a summary of AI decisions vs outcomes.

## Resource Impact

| Resource | Before | After | Delta |
|----------|--------|-------|-------|
| RAM | baseline | baseline | 0 MB (in-memory prompt change only) |
| Storage | baseline | baseline | 0 KB (no new tables) |
| CPU | baseline | baseline | 0% (same number of API calls) |
| API calls | baseline | baseline | Same — one DeepSeek + one Tavily per signal |
| Latency | baseline | baseline + 10ms | Trader stats query is <1ms |
