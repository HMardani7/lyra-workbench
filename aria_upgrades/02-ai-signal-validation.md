# AI Signal Validation — Implementation Plan

> **For Hermes:** Use subagent-driven-development skill to implement this plan task-by-task.

**Goal:** Give Aria an AI-powered research layer that validates signals before paper-trading them. When a signal arrives, Aria researches current market conditions (news, sentiment, price action) via DeepSeek V4 Pro + web search and decides whether the trade is worth copying.

**Architecture:** Tick-based non-blocking research with SQLite cache. Signals enter "pending_validation" status on first tick, research runs async (up to 120s), and on next tick(s) the verdict is applied. Multiple coins research in parallel via `asyncio.gather`. Cache hits (same coin within 15 min) skip the LLM entirely.

**Tech Stack:** Python 3.11+, asyncio, httpx, SQLModel, Pydantic Settings, pytest, SQLite

---

## What This Upgrade Does

### The Problem It Solves

Aria currently copies trades blindly — if a wallet opens a position, Aria copies it. There's no intelligence layer asking _"is this actually a good trade right now?"_

**Real examples of trades Aria would copy without this upgrade:**

| Signal | Why It's Bad | Result Without AI |
|--------|-------------|-------------------|
| ETH long | SEC just sued a major DeFi protocol using ETH. Price about to dump 15%. | Aria goes long, loses money |
| SOL long | Network is congested, 75% of transactions failing. Community outrage on Twitter. | Aria ignores the red flags |
| DOGE long | Elon tweeted. Pure hype pump, already up 40% in 1 hour. Chasing the top. | Aria FOMOs in at the peak |
| BTC long | Multiple positive catalysts: ETF inflows +$1B, halving approaching, institutional adoption | ✅ This one's actually fine |

**Root cause:** Aria has no way to distinguish between a smart trade and a dumb one. All signals look the same — wallet X opened position Y. The existing rules (confidence score, exposure limits, duplicate cooldown) are mechanical filters. They don't understand _context_.

### What This Upgrade Changes

Adds an **AI Signal Validator** — a research step between the duplicate cooldown check and exposure rules:

```
Signal → confidence check → duplicate cooldown 
       → 🔍 AI VALIDATION (NEW) → exposure rules → TRADE
```

**How it works:**

1. Signal arrives during `process_all_signals()` tick
2. If coin isn't in cache (last 15 min) → mark `pending_validation`, fire async research
3. Research: DeepSeek V4 Pro + Tavily Search API looks up current news, sentiment, price trends
4. LLM returns structured verdict: `{approved: bool, confidence: 0-1, reason, key_factors}`
5. Next tick: verdict is in cache → signal proceeds (approved) or gets ignored (rejected)
6. Cache hit: same coin within 15 min → instant verdict, zero API cost

**Critical design decision — non-blocking:** The signal pipeline never waits for research. A signal marked `pending_validation` just sits until the next tick. This means:

- 3 signals for different coins in one tick → all 3 research in parallel
- 8 signals for the same coin → 1 research call, 7 cache hits next tick
- Research timeout → signal expires to "ignored" (safer than approving blind)

### Risk It Prevents

- **Narrative FOMO:** Copying a trade just because it's popular, not because conditions are good
- **Adverse events:** Opening a position right as negative news breaks (SEC actions, hacks, network outages)
- **Pump chasing:** Copying trades that are already overextended, entering at the local top
- **Uninformed copying:** Copying a wallet that's making a contrarian bet without understanding why

### How It Improves Profitability

By filtering out bad-context trades while still copying good ones:

- Fewer losing trades from adverse news events
- Avoids entering positions during market turbulence
- Only copies when research backs up the signal direction
- Cache makes good signals FAST (instant on repeat coins), no delay for quality setups

### Configurability

| Setting | Default | What It Does |
|---------|---------|--------------|
| `AI_VALIDATION_ENABLED` | `true` | Master switch. Set to `false` to disable entirely. |
| `AI_VALIDATION_MODEL` | `deepseek/deepseek-v4-pro` | Model ID on OpenRouter. Can swap to flash for speed. |
| `AI_VALIDATION_TIMEOUT_SECONDS` | `120` | Max time for research. Signal expires if exceeded. |
| `AI_VALIDATION_CACHE_TTL_SECONDS` | `900` | 15 min. Same-coin signals within this window hit cache. |
| `AI_VALIDATION_FALLBACK_ON_TIMEOUT` | `false` | If true, approve signal when research times out. Safer to leave false. |
| `AI_VALIDATION_MIN_CONFIDENCE` | `0.6` | LLM must return confidence ≥ this to approve. |
| `TAVILY_API_KEY` | (required) | API key for Tavily. Free tier: 1,000 queries/month. |

### Scope

- Applies to **both** Hyperliquid-detected signals and Invo webhook signals
- Researches per **coin** (not per signal) — 5 wallets signaling BTC = 1 research call
- **Does not** override exposure limits or risk rules — it's an additional filter, not a replacement
- **Does not** close existing trades — validation only gates new opens
- **Does not** require more RAM or storage — SQLite cache is ~1MB, auto-prunes stale entries

---

## Background

### System Constraints

Aria runs on a small VPS (4GB RAM, 80GB storage) alongside other services. This architecture is designed for minimal resource impact:

- **No new services or processes** — everything runs inside Aria's existing Python process
- **No vector database** — SQLite cache is lighter and doesn't need embeddings
- **No ML model loaded in memory** — all AI work is external API calls
- **Dependencies:** `httpx` for async HTTP (likely already in use by Aria), `asyncio` from stdlib
- **Storage:** ~1MB/year for validation cache. Auto-pruned every tick so it never grows unboundedly.

### Why Tick-Based Async Instead of Blocking

The naive approach — block the signal pipeline while researching — has two fatal problems:

1. **3 different coins need research → 3 × 60s = 3 minutes delay.** The pipeline stalls.
2. **What if research fails?** Do we lose the signal? Hold it in memory hoping the process doesn't crash?

Tick-based async solves both:
- Research starts immediately but the pipeline continues processing other signals
- Results persist in SQLite — survive process restarts
- Failed/timeout research → signal naturally expires to "ignored" on next tick

### DeepSeek V4 Pro Capabilities

| Spec | Value | Relevance |
|------|-------|-----------|
| 1.6T params (49B active) | MoE architecture | Strong reasoning for market analysis |
| 1M token context | Huge window | Can ingest lots of search results |
| Tool calling | ✅ Supported | Tavily Search API passed as a tool |
| Response format | JSON mode supported | Structured verdict output |
| Pricing | $0.435/M input, $0.87/M output | ~$0.002 per research call |

The model does NOT have built-in web search — we pass Tavily as a function calling tool. The LLM decides when to search and what to search for, then synthesizes results into a verdict.

---

## Files to Touch

| File | Action | Purpose |
|------|--------|---------|
| `app/core/config.py` | Modify | Add AI validation config settings |
| `.env.example` | Modify | Add `AI_VALIDATION_*` and `TAVILY_API_KEY` |
| `app/db/models.py` | Modify | Add `AIValidationCache` model |
| `app/ai/__init__.py` | Create | New `ai` package |
| `app/ai/researcher.py` | Create | Core research logic (prompt, LLM call, cache) |
| `app/ai/tavily_search.py` | Create | Tavily Search API client |
| `app/ai/validator.py` | Create | `check_ai_validation()` entry point |
| `app/risk/rules.py` | Modify | Call `check_ai_validation()` in `check_open_rules()` |
| `app/invo/processor.py` | Modify | Call `check_ai_validation()` in `_process_open()` |
| `tests/test_ai_validation.py` | Create | Tests for validation logic |

---

## Task 1: Add config settings

**Objective:** Add all AI validation settings to the config model.

**Files:**
- Modify: `app/core/config.py`
- Modify: `.env.example`

**Step 1: Add fields to Settings class**

In `app/core/config.py`, add after the duplicate cooldown settings (around line 108):

```python
    # AI Signal Validation
    ai_validation_enabled: bool = True
    ai_validation_model: str = "deepseek/deepseek-v4-pro"
    ai_validation_timeout_seconds: int = 120
    ai_validation_cache_ttl_seconds: int = 900  # 15 minutes
    ai_validation_fallback_on_timeout: bool = False
    ai_validation_min_confidence: float = 0.6
    ai_validation_openrouter_api_key: str = ""
    tavily_api_key: str = ""
```

**Step 2: Add to .env.example**

After the `DUPLICATE_TRADE_COOLDOWN_SECONDS` line:

```
# AI Signal Validation
AI_VALIDATION_ENABLED=true
AI_VALIDATION_MODEL=deepseek/deepseek-v4-pro
AI_VALIDATION_TIMEOUT_SECONDS=120
AI_VALIDATION_CACHE_TTL_SECONDS=900
AI_VALIDATION_FALLBACK_ON_TIMEOUT=false
AI_VALIDATION_MIN_CONFIDENCE=0.6
AI_VALIDATION_OPENROUTER_API_KEY=
TAVILY_API_KEY=
```

**Step 3: Verify**

```bash
cd /workspace/repos/hyperliquid-copy-bot
uv run python -c "
from app.core.config import get_settings
s = get_settings()
print(f'ai_validation_enabled: {s.ai_validation_enabled}')
print(f'ai_validation_model: {s.ai_validation_model}')
print(f'ai_validation_timeout: {s.ai_validation_timeout_seconds}s')
print(f'cache_ttl: {s.ai_validation_cache_ttl_seconds}s')
print(f'min_confidence: {s.ai_validation_min_confidence}')
"
```

Expected: All values print with defaults.

---

## Task 2: Create AIValidationCache DB model

**Objective:** Add a cache table for AI validation results. Lightweight — auto-pruned on every tick.

**Files:**
- Modify: `app/db/models.py`

**Step 1: Add model**

In `app/db/models.py`, add after the `AuditLog` model (around line 280):

```python
class AIValidationCache(SQLModel, table=True):
    """Cached AI validation results. Pruned on every tick when stale."""

    __tablename__ = "ai_validation_cache"

    id: int | None = Field(default=None, primary_key=True)
    coin: str = Field(index=True)
    side: str = "long"  # long/short — used for cache key differentiation
    approved: bool = False
    confidence: float = 0.0
    reason: str = ""
    key_factors: str = "[]"  # JSON list of strings
    researched_at: datetime = Field(default_factory=utc_now)
```

**Step 2: Verify model creation**

```bash
cd /workspace/repos/hyperliquid-copy-bot
uv run python -c "
from app.db.init_db import init_database
from app.db.models import AIValidationCache
from app.db.session import get_session
from sqlmodel import select

init_database()
with get_session() as session:
    count = len(session.exec(select(AIValidationCache)).all())
    print(f'Cache rows: {count} (should be 0)')
print('Model created OK')
"
```

Expected: `Cache rows: 0` and no errors.

---

## Task 3: Implement Tavily Search API client

**Objective:** Create a lightweight Tavily client that DeepSeek V4 Pro calls as a tool.

**Files:**
- Create: `app/ai/__init__.py`
- Create: `app/ai/tavily_search.py`

**Step 1: Create package init**

`app/ai/__init__.py`:
```python
"""AI signal validation module."""
```

**Step 2: Create Tavily client**

`app/ai/tavily_search.py`:
```python
"""Tavily Search API client — lightweight, async, no heavy deps.

Tavily is an AI-optimized search API. Results come pre-summarized,
which is ideal for feeding into an LLM for signal validation.
Free tier: 1,000 queries/month.
"""

from __future__ import annotations

import asyncio
from typing import Any

import httpx

from app.core.config import get_settings


TAVILY_API_URL = "https://api.tavily.com/search"


async def tavily_search(query: str, count: int = 5) -> list[dict[str, Any]]:
    """Search the web via Tavily Search API.

    Tavily returns AI-optimized results with title, url, and content
    (summary) fields. Uses POST with API key in the request body.

    Returns a list of simplified result dicts with 'title', 'url',
    'content' keys.
    """
    settings = get_settings()
    api_key = settings.tavily_api_key

    if not api_key:
        return [{"title": "Error", "url": "", "content": "Tavily API key not configured — set TAVILY_API_KEY in .env"}]

    payload: dict[str, Any] = {
        "api_key": api_key,
        "query": query,
        "search_depth": "basic",
        "max_results": min(count, 10),
        "include_answer": False,
    }

    async with httpx.AsyncClient(timeout=15.0) as client:
        try:
            response = await client.post(TAVILY_API_URL, json=payload)
            response.raise_for_status()
            data = response.json()
        except httpx.HTTPError as e:
            return [{"title": "Search Error", "url": "", "content": str(e)}]

    results: list[dict[str, Any]] = []
    for item in data.get("results", [])[:count]:
        results.append({
            "title": item.get("title", ""),
            "url": item.get("url", ""),
            "content": item.get("content", ""),
        })

    if not results:
        return [{"title": "No Results", "url": "", "content": f"No results found for: {query}"}]

    return results


def tavily_search_sync(query: str, count: int = 5) -> list[dict[str, Any]]:
    """Synchronous wrapper for tavily_search. Used when calling from non-async context."""
    return asyncio.run(tavily_search(query, count))
```

**Step 3: Verify**

```bash
cd /workspace/repos/hyperliquid-copy-bot
uv run python -c "
import asyncio
from app.ai.tavily_search import tavily_search
# Test with no API key — should return error gracefully
result = asyncio.run(tavily_search('Bitcoin price today'))
print(f'Result count: {len(result)}')
print(f'First result: {result[0]}')
print('OK - no crash without API key')
"
```

---

## Task 4: Implement the core researcher

**Objective:** The research engine that calls DeepSeek V4 Pro with Tavily as a tool, gets a structured verdict back, and caches it.

**Files:**
- Create: `app/ai/researcher.py`

**Step 1: Create researcher module**

`app/ai/researcher.py`:
```python
"""AI researcher — calls DeepSeek V4 Pro with web search to validate signals."""

from __future__ import annotations

import asyncio
import json
from datetime import UTC, datetime, timedelta

import httpx
from loguru import logger
from sqlmodel import select

from app.core.config import get_settings
from app.db.models import AIValidationCache
from app.db.session import get_session


# In-memory set of coins currently being researched (prevents duplicate
# parallel calls for the same coin). Cleared on process restart, which
# is fine — cache takes over on restart.
_pending_research: set[str] = set()

# OpenRouter API
OPENROUTER_API_URL = "https://openrouter.ai/api/v1/chat/completions"

# The system prompt sent to DeepSeek V4 Pro
SYSTEM_PROMPT = """You are a crypto trading signal validator. Your job is to research
current market conditions for a cryptocurrency and decide whether to copy a trade signal.

You have access to a web_search tool. Use it to research recent news, price action,
sentiment, and any relevant events for the coin.

Rules:
- Be objective. Don't approve a trade just because it exists.
- Consider: recent price trends (1h, 24h), major news events, social sentiment,
  on-chain activity, market-wide conditions.
- If there are clear negative signals (SEC action, hack, network outage, extreme
  overbought conditions), reject the trade.
- If conditions are mixed but no major red flags, approve with moderate confidence.
- If conditions are clearly favorable (positive catalysts, strong momentum, no
  negative news), approve with high confidence.
- Be concise. The user's bot is waiting for your verdict.

Respond with a JSON object:
{
  "approved": true/false,
  "confidence": 0.0-1.0,
  "reason": "Brief explanation of your decision",
  "key_factors": ["factor 1", "factor 2"]
}"""

# The user prompt template
USER_PROMPT_TEMPLATE = """Research {coin} and determine whether a {side} position
should be opened at approximately ${price:.2f}.

The trade signal came from a tracked wallet on Hyperliquid. The bot will copy this
trade as a spot long on Bitget if you approve.

Research current conditions for {coin} and return your verdict."""

# Tool definition passed to the LLM
TAVILY_SEARCH_TOOL = {
    "type": "function",
    "function": {
        "name": "web_search",
        "description": "Search the web for current information about a topic. Use for news, price data, sentiment, and events.",
        "parameters": {
            "type": "object",
            "properties": {
                "query": {
                    "type": "string",
                    "description": "The search query. Be specific. Examples: 'Bitcoin price news today', 'Ethereum SEC regulation 2026', 'Solana network outage status'"
                }
            },
            "required": ["query"]
        }
    }
}


async def research_coin(coin: str, side: str, price: float) -> ValidationResult:
    """Research a coin and return a validation verdict.

    This is the main entry point. It handles cache lookup, LLM call,
    and result storage.
    """
    settings = get_settings()

    # Check cache first
    cached = _lookup_cache(coin, side)
    if cached:
        return cached

    # Mark as pending to prevent duplicate parallel calls
    cache_key = f"{coin}:{side}"
    if cache_key in _pending_research:
        # Another coroutine is already researching this coin.
        # Return a "pending" result — caller should retry next tick.
        return ValidationResult(
            approved=False,
            confidence=0.0,
            reason="Already being researched — retry next tick",
            key_factors=[],
            pending=True,
        )
    _pending_research.add(cache_key)

    try:
        result = await _call_llm_with_search(coin, side, price, settings)
    except asyncio.TimeoutError:
        logger.warning(f"AI validation timed out for {coin} {side}")
        if settings.ai_validation_fallback_on_timeout:
            result = ValidationResult(
                approved=True,
                confidence=0.5,
                reason="Research timed out — defaulting to approve",
                key_factors=["timeout fallback"],
            )
        else:
            result = ValidationResult(
                approved=False,
                confidence=0.0,
                reason=f"Research timed out after {settings.ai_validation_timeout_seconds}s",
                key_factors=["timeout"],
            )
    except Exception as e:
        logger.error(f"AI validation error for {coin} {side}: {e}")
        result = ValidationResult(
            approved=False,
            confidence=0.0,
            reason=f"Research error: {e}",
            key_factors=["error"],
        )
    finally:
        _pending_research.discard(cache_key)

    # Store in cache
    _store_cache(coin, side, result)

    return result


async def _call_llm_with_search(
    coin: str, side: str, price: float, settings
) -> ValidationResult:
    """Call DeepSeek V4 Pro via OpenRouter with Tavily as a tool.

    Uses a multi-turn approach: send prompt + tool definition, the LLM
    calls web_search, we execute the search, send results back, and the
    LLM returns the final verdict.
    """
    from app.ai.tavily_search import tavily_search

    api_key = settings.ai_validation_openrouter_api_key
    if not api_key:
        return ValidationResult(
            approved=False,
            confidence=0.0,
            reason="OpenRouter API key not configured (AI_VALIDATION_OPENROUTER_API_KEY)",
            key_factors=["no_api_key"],
        )

    headers = {
        "Authorization": f"Bearer {api_key}",
        "Content-Type": "application/json",
        "HTTP-Referer": "https://aria-copybot.local",
        "X-Title": "Aria Copy Bot",
    }

    user_prompt = USER_PROMPT_TEMPLATE.format(coin=coin, side=side, price=price)

    # First call: send the prompt with web_search tool
    payload = {
        "model": settings.ai_validation_model,
        "messages": [
            {"role": "system", "content": SYSTEM_PROMPT},
            {"role": "user", "content": user_prompt},
        ],
        "tools": [TAVILY_SEARCH_TOOL],
        "tool_choice": "auto",
        "temperature": 0.3,  # Low temp for consistent analysis
        "max_tokens": 2000,
    }

    async with httpx.AsyncClient(timeout=settings.ai_validation_timeout_seconds) as client:
        # Turn 1: LLM decides what to search
        response1 = await client.post(OPENROUTER_API_URL, json=payload, headers=headers)
        response1.raise_for_status()
        data1 = response1.json()

        choice1 = data1["choices"][0]
        message1 = choice1["message"]

        # If the LLM called the web_search tool, execute it
        tool_calls = message1.get("tool_calls", [])
        search_results_text = ""

        if tool_calls:
            # Append assistant message with tool calls
            messages = [
                {"role": "system", "content": SYSTEM_PROMPT},
                {"role": "user", "content": user_prompt},
                message1,  # Assistant message with tool_calls
            ]

            # Execute each search
            for tc in tool_calls:
                if tc["function"]["name"] == "web_search":
                    args = json.loads(tc["function"]["arguments"])
                    query = args.get("query", f"{coin} crypto news")
                    results = await tavily_search(query, count=5)
                    search_results_text = json.dumps(results, indent=2)

                    messages.append({
                        "role": "tool",
                        "tool_call_id": tc["id"],
                        "content": search_results_text,
                    })

            # Turn 2: LLM synthesizes results into verdict
            payload2 = {
                "model": settings.ai_validation_model,
                "messages": messages,
                "response_format": {"type": "json_object"},
                "temperature": 0.2,
                "max_tokens": 1000,
            }
            response2 = await client.post(OPENROUTER_API_URL, json=payload2, headers=headers)
            response2.raise_for_status()
            data2 = response2.json()

            verdict_text = data2["choices"][0]["message"]["content"]
        else:
            # LLM didn't call search — use its direct response
            verdict_text = message1.get("content", "")

    # Parse verdict
    return _parse_verdict(verdict_text, coin, side)


def _parse_verdict(text: str, coin: str, side: str) -> ValidationResult:
    """Parse the LLM's JSON verdict into a ValidationResult."""
    try:
        # Extract JSON from response (may have markdown wrapping)
        import re
        json_match = re.search(r'\{[\s\S]*\}', text)
        if json_match:
            data = json.loads(json_match.group(0))
        else:
            data = json.loads(text)

        return ValidationResult(
            approved=data.get("approved", False),
            confidence=float(data.get("confidence", 0.0)),
            reason=data.get("reason", "No reason provided"),
            key_factors=data.get("key_factors", []),
        )
    except (json.JSONDecodeError, ValueError) as e:
        logger.error(f"Failed to parse LLM verdict for {coin}: {e}\nRaw: {text[:500]}")
        return ValidationResult(
            approved=False,
            confidence=0.0,
            reason=f"Failed to parse research verdict: {e}",
            key_factors=["parse_error"],
        )


def _lookup_cache(coin: str, side: str) -> ValidationResult | None:
    """Check if a cached result exists and is still fresh."""
    settings = get_settings()
    cutoff = datetime.now(UTC) - timedelta(seconds=settings.ai_validation_cache_ttl_seconds)

    with get_session() as session:
        cached = session.exec(
            select(AIValidationCache)
            .where(
                AIValidationCache.coin == coin,
                AIValidationCache.side == side,
                AIValidationCache.researched_at >= cutoff,
            )
            .order_by(AIValidationCache.researched_at.desc())  # type: ignore[arg-type]
        ).first()

    if cached:
        logger.info(f"Cache HIT for {coin} {side}: approved={cached.approved}")
        return ValidationResult(
            approved=cached.approved,
            confidence=cached.confidence,
            reason=cached.reason,
            key_factors=json.loads(cached.key_factors),
        )
    return None


def _store_cache(coin: str, side: str, result: ValidationResult) -> None:
    """Store a validation result in the cache."""
    with get_session() as session:
        cache = AIValidationCache(
            coin=coin,
            side=side,
            approved=result.approved,
            confidence=result.confidence,
            reason=result.reason,
            key_factors=json.dumps(result.key_factors),
        )
        session.add(cache)
        session.commit()
    logger.info(f"Cache STORED for {coin} {side}: approved={result.approved}")


def prune_stale_cache() -> int:
    """Delete cache entries older than TTL. Call on every tick.

    Returns count of deleted rows.
    """
    settings = get_settings()
    cutoff = datetime.now(UTC) - timedelta(seconds=settings.ai_validation_cache_ttl_seconds)

    with get_session() as session:
        stmt = select(AIValidationCache).where(AIValidationCache.researched_at < cutoff)
        stale = list(session.exec(stmt).all())
        count = len(stale)
        for entry in stale:
            session.delete(entry)
        session.commit()

    if count > 0:
        logger.debug(f"Pruned {count} stale cache entries")
    return count


class ValidationResult:
    """The result of AI signal validation."""

    def __init__(
        self,
        approved: bool,
        confidence: float,
        reason: str,
        key_factors: list[str],
        pending: bool = False,
    ):
        self.approved = approved
        self.confidence = confidence
        self.reason = reason
        self.key_factors = key_factors
        self.pending = pending

    def __repr__(self) -> str:
        return (
            f"ValidationResult(approved={self.approved}, "
            f"confidence={self.confidence:.2f}, reason={self.reason!r})"
        )
```

**Step 2: Verify imports compile**

```bash
cd /workspace/repos/hyperliquid-copy-bot
uv run python -c "
from app.ai.researcher import (
    ValidationResult,
    research_coin,
    prune_stale_cache,
    SYSTEM_PROMPT,
    USER_PROMPT_TEMPLATE,
    TAVILY_SEARCH_TOOL,
)
print('All imports OK')
print(f'System prompt length: {len(SYSTEM_PROMPT)} chars')
print(f'Tool name: {TAVILY_SEARCH_TOOL[\"function\"][\"name\"]}')
"
```

---

## Task 5: Implement the validator entry point

**Objective:** Create `check_ai_validation()` — the function that `check_open_rules()` and Invo processor call. This is a synchronous wrapper around the async researcher, designed to be called from Aria's tick loop.

**Files:**
- Create: `app/ai/validator.py`

**Step 1: Create validator module**

`app/ai/validator.py`:
```python
"""AI validation entry point — called from signal processing pipeline."""

from __future__ import annotations

import asyncio

from loguru import logger

from app.core.config import get_settings
from app.ai.researcher import (
    ValidationResult,
    _lookup_cache,
    research_coin,
)


# Track which coins we've already fired research for in this tick
# (prevents multiple signals for the same coin from all firing research)
_tick_fired: dict[str, bool] = {}


def check_ai_validation(coin: str, side: str, price: float) -> tuple[bool, str]:
    """Check if a signal passes AI validation.

    Called from check_open_rules() and Invo processor.

    Three possible outcomes:
    1. Cache HIT → returns (approved, reason) immediately
    2. Cache MISS → fires async research, returns (False, "pending...")
       Signal will be re-checked next tick.
    3. AI disabled → returns (True, "")

    Returns (should_proceed, reason).
    """
    settings = get_settings()

    if not settings.ai_validation_enabled:
        return True, ""

    # Check cache first
    cached = _lookup_cache(coin, side)
    if cached:
        if cached.approved and cached.confidence >= settings.ai_validation_min_confidence:
            return True, f"AI approved ({cached.confidence:.0%}): {cached.reason}"
        elif cached.approved:
            return False, (
                f"AI confidence too low ({cached.confidence:.0%} < "
                f"{settings.ai_validation_min_confidence:.0%}): {cached.reason}"
            )
        else:
            return False, f"AI rejected ({cached.confidence:.0%}): {cached.reason}"

    # Cache miss — fire research (non-blocking)
    cache_key = f"{coin}:{side}"
    if cache_key not in _tick_fired:
        _tick_fired[cache_key] = True
        _fire_research_background(coin, side, price)

    return False, f"AI research pending for {coin} — check next tick"


def _fire_research_background(coin: str, side: str, price: float) -> None:
    """Fire research in the background. Non-blocking — does not wait for result.

    Uses asyncio.ensure_future() to schedule research. Results are stored
    in the cache and picked up on the next tick.
    """
    try:
        loop = asyncio.get_event_loop()
        if loop.is_running():
            # We're inside an async context — schedule the coroutine
            loop.create_task(research_coin(coin, side, price))
            logger.info(f"Fired async research for {coin} {side} @ ${price:.2f}")
        else:
            # Running from sync context — spin up research in a thread
            import threading
            t = threading.Thread(
                target=lambda: asyncio.run(research_coin(coin, side, price)),
                daemon=True,
            )
            t.start()
            logger.info(f"Fired threaded research for {coin} {side} @ ${price:.2f}")
    except RuntimeError:
        # No event loop — use thread
        import threading
        t = threading.Thread(
            target=lambda: asyncio.run(research_coin(coin, side, price)),
            daemon=True,
        )
        t.start()
        logger.info(f"Fired threaded research for {coin} {side} @ ${price:.2f}")


def reset_tick_state() -> None:
    """Clear per-tick tracking. Call at the start of each process_all_signals()."""
    _tick_fired.clear()
```

**Step 2: Verify**

```bash
cd /workspace/repos/hyperliquid-copy-bot
uv run python -c "
from app.ai.validator import check_ai_validation, reset_tick_state
print('validator imports OK')

# Test with AI disabled (no API key)
# Should return (True, '') since cache is empty + no research being fired without API key
reset_tick_state()
result, reason = check_ai_validation('BTC', 'long', 95000.0)
print(f'Result: {result}, Reason: {reason}')
print('NOTE: result may be True or False depending on config — this is expected')
"
```

---

## Task 6: Integrate into check_open_rules()

**Objective:** Call `check_ai_validation()` in the Hyperliquid signal pipeline. Insert after the duplicate cooldown check (from upgrade #1) and before exposure checks.

**Files:**
- Modify: `app/risk/rules.py`

**Step 1: Add the AI validation check**

In `check_open_rules()`, after the duplicate cooldown check and before the price check (around line 45), add:

```python
    # AI signal validation
    can_open, reason = check_ai_validation(signal.coin, signal.side, signal.price)
    if not can_open and "pending" not in reason.lower():
        return False, reason, 0.0
    # If "pending", let it fall through — signal waits for next tick
    # (the signal processor will re-pick it up since processed_status stays "new")
```

Also add the import at the top of the file:

```python
from app.ai.validator import check_ai_validation
```

The full `check_open_rules()` after all upgrades should look like:

```python
def check_open_rules(
    signal: DetectedSignal,
    portfolio_id: int,
    balance: float,
    baseline_ts_ms: int | None,
) -> tuple[bool, str, float]:
    """Check if a signal should open a paper trade."""
    settings = get_settings()

    # Must be live-compatible
    if not signal.live_compatible_bitget_spot:
        if settings.paper_spot_require_live_compatible:
            return False, f"Not live-compatible: {signal.compatibility_reason}", 0.0

    # Must be open_long or increase_long
    if signal.event_type not in ("open_long", "increase_long"):
        return False, f"Event type {signal.event_type} not an open signal", 0.0

    # Must have sufficient confidence
    if signal.confidence_score < settings.min_signal_confidence:
        return False, f"Low confidence: {signal.confidence_score:.2f}", 0.0

    # Duplicate trade cooldown (upgrade #1)
    can_open, reason = check_duplicate_cooldown(
        signal.coin, cooldown_seconds=settings.duplicate_trade_cooldown_seconds
    )
    if not can_open:
        return False, reason, 0.0

    # AI signal validation (upgrade #2)
    can_open, reason = check_ai_validation(signal.coin, signal.side, signal.price)
    if not can_open and "pending" not in reason.lower():
        return False, reason, 0.0
    # "pending" signals stay as "new" and are re-evaluated next tick

    # Must have a price
    if signal.price <= 0:
        return False, "No price available", 0.0

    # ... rest of existing checks (baseline, add-to-position, exposure, etc.)
```

**Step 2: Handle "pending" in signal_processor.py**

In `app/paper/signal_processor.py`, modify `process_all_signals()` to handle the "pending" case. When `check_open_rules()` returns `(False, "AI research pending...")`, the signal should NOT be marked as "ignored" — it should stay "new" so it gets re-processed next tick.

```python
# In process_all_signals(), around line 52, after process_signal_for_spot():
for signal in signals:
    try:
        result = process_signal_for_spot(signal)
        if result.action == "pending_validation":
            # Don't mark as processed — check again next tick
            continue
        _mark_signal(signal, result.action, result.reason)
        _send_alert(signal, result)
        counts[result.action] = counts.get(result.action, 0) + 1
    except Exception as e:
        logger.error(f"Error processing signal {signal.id}: {e}")
        _mark_signal(signal, "error", str(e))
        counts["error"] += 1
```

And in `process_signal_for_spot()`, return `LedgerResult("pending_validation", reason)` when AI research is pending.

**Step 3: Add reset_tick_state() call**

In `process_all_signals()`, at the very beginning, add:

```python
def process_all_signals() -> dict[str, int]:
    """Process signals through the SPOT ledger and send alerts."""
    from app.ai.validator import reset_tick_state
    from app.ai.researcher import prune_stale_cache
    from app.paper.execution import process_signal_for_spot

    reset_tick_state()  # Clear per-tick research tracking
    prune_stale_cache()  # Remove expired cache entries

    # ... rest of existing code
```

Also add `"pending_validation": 0` to the `counts` dict.

**Step 4: Verify compilation**

```bash
cd /workspace/repos/hyperliquid-copy-bot
uv run python -c "
from app.risk.rules import check_open_rules
from app.ai.validator import check_ai_validation
from app.ai.researcher import prune_stale_cache
print('All integrated imports OK')
"
```

---

## Task 7: Integrate into Invo processor

**Objective:** Add the same AI validation check to the Invo webhook pipeline.

**Files:**
- Modify: `app/invo/processor.py`

**Step 1: Add AI check in `_process_open()`**

After the duplicate cooldown check (from upgrade #1) and before the position_key check, add:

```python
    # AI signal validation
    from app.ai.validator import check_ai_validation

    can_open, reason = check_ai_validation(
        payload.symbol, payload.side, payload.entry_price or 0.0
    )
    if not can_open and "pending" not in reason.lower():
        _update_signal(signal, "ignored", "ignored", reason)
        _notify_ignored(payload, reason)
        return InvoWebhookResponse(accepted=True, decision="ignored", reason=reason)
    # If "pending", the webhook response still needs to acknowledge.
    # Return accepted but with a "pending_validation" decision.
    if not can_open:
        _update_signal(signal, "processing", "pending_validation", reason)
        return InvoWebhookResponse(
            accepted=True, decision="pending_validation", reason=reason
        )
```

**Step 2: Verify**

```bash
cd /workspace/repos/hyperliquid-copy-bot
uv run python -c "
from app.invo.processor import process_invo_signal
print('Invo processor with AI validation imports OK')
"
```

---

## Task 8: Add Telegram notifications for AI rejections

**Objective:** When AI rejects a signal, send a Telegram alert so the user knows why.

**Files:**
- Modify: `app/paper/signal_processor.py` (or `app/telegram/messages.py`)

**Step 1: Add AI rejection alert**

In `_send_alert()` in `signal_processor.py`, add a case for AI rejections:

```python
def _send_alert(signal: DetectedSignal, result: LedgerResult) -> None:
    # ... existing code ...
    
    if result.action == "ai_rejected":
        emoji = "🤖"
    elif result.action == "opened":
        emoji = "✅"
    # ... rest of existing emoji mapping ...
    
    # Format AI rejection message
    if result.action == "ai_rejected":
        msg = (
            f"{emoji} <b>AI REJECTED SIGNAL</b>\n"
            f"Wallet: {short_address(signal.wallet_address)}\n"
            f"Coin: {signal.coin} | {signal.event_type}\n"
            f"Price: ${signal.price:.2f}\n"
            f"Reason: {result.reason}"
        )
        send_message_sync(msg)
        return
```

---

## Task 9: Write tests

**Objective:** Unit tests for cache, validation logic, and integration.

**Files:**
- Create: `tests/test_ai_validation.py`

**Step 1: Create test file**

```python
"""Tests for AI signal validation."""

import json
from datetime import UTC, datetime, timedelta

import pytest
from sqlmodel import select

from app.ai.researcher import (
    ValidationResult,
    _lookup_cache,
    _store_cache,
    _parse_verdict,
    prune_stale_cache,
)
from app.db.models import AIValidationCache
from app.db.session import get_session
from app.core.config import Settings


def make_settings(**overrides) -> Settings:
    """Build a Settings instance with overrides for testing."""
    return Settings(
        ai_validation_enabled=True,
        ai_validation_cache_ttl_seconds=900,
        ai_validation_min_confidence=0.6,
        ai_validation_timeout_seconds=120,
        ai_validation_fallback_on_timeout=False,
        ai_validation_model="deepseek/deepseek-v4-pro",
        **overrides,
    )


class TestParseVerdict:
    """Tests for _parse_verdict()."""

    def test_valid_approved_verdict(self):
        text = json.dumps({
            "approved": True,
            "confidence": 0.85,
            "reason": "Strong momentum, ETF inflows positive",
            "key_factors": ["ETF inflows $500M", "broke resistance"],
        })
        result = _parse_verdict(text, "BTC", "long")
        assert result.approved is True
        assert result.confidence == 0.85
        assert "ETF" in result.reason
        assert len(result.key_factors) == 2

    def test_valid_rejected_verdict(self):
        text = json.dumps({
            "approved": False,
            "confidence": 0.9,
            "reason": "SEC enforcement action announced",
            "key_factors": ["SEC lawsuit filed", "price dumping"],
        })
        result = _parse_verdict(text, "ETH", "long")
        assert result.approved is False
        assert result.confidence == 0.9
        assert "SEC" in result.reason

    def test_markdown_wrapped_json(self):
        text = '```json\n{"approved": true, "confidence": 0.7, "reason": "ok", "key_factors": ["a"]}\n```'
        result = _parse_verdict(text, "BTC", "long")
        assert result.approved is True
        assert result.confidence == 0.7

    def test_malformed_json(self):
        text = "not json at all"
        result = _parse_verdict(text, "BTC", "long")
        assert result.approved is False
        assert "parse" in result.reason.lower()

    def test_missing_fields(self):
        text = '{"approved": true}'
        result = _parse_verdict(text, "BTC", "long")
        assert result.approved is True
        assert result.confidence == 0.0  # default


class TestCacheOperations:
    """Tests for cache store/lookup/prune."""

    @pytest.fixture(autouse=True)
    def setup_db(self):
        """Initialize DB and clean up after test."""
        from app.db.init_db import init_database
        init_database()

        # Clean any existing cache
        with get_session() as session:
            for entry in session.exec(select(AIValidationCache)).all():
                session.delete(entry)
            session.commit()

        yield

        with get_session() as session:
            for entry in session.exec(select(AIValidationCache)).all():
                session.delete(entry)
            session.commit()

    def test_store_and_lookup(self, monkeypatch):
        """Store a result and verify it's retrievable."""
        # Fix cache TTL
        monkeypatch.setattr(
            "app.ai.researcher.get_settings",
            lambda: make_settings(ai_validation_cache_ttl_seconds=900),
        )

        result = ValidationResult(
            approved=True,
            confidence=0.8,
            reason="Good conditions",
            key_factors=["factor1", "factor2"],
        )
        _store_cache("BTC", "long", result)

        cached = _lookup_cache("BTC", "long")
        assert cached is not None
        assert cached.approved is True
        assert cached.confidence == 0.8
        assert "factor1" in cached.key_factors

    def test_cache_hit_same_coin_side(self, monkeypatch):
        """Same coin+side should return cached result."""
        monkeypatch.setattr(
            "app.ai.researcher.get_settings",
            lambda: make_settings(ai_validation_cache_ttl_seconds=900),
        )

        result = ValidationResult(approved=False, confidence=0.9, reason="Reject", key_factors=[])
        _store_cache("SOL", "long", result)

        cached = _lookup_cache("SOL", "long")
        assert cached is not None
        assert cached.approved is False

    def test_cache_miss_different_coin(self, monkeypatch):
        """Different coin should miss cache."""
        monkeypatch.setattr(
            "app.ai.researcher.get_settings",
            lambda: make_settings(ai_validation_cache_ttl_seconds=900),
        )

        result = ValidationResult(approved=True, confidence=0.7, reason="ok", key_factors=[])
        _store_cache("BTC", "long", result)

        cached = _lookup_cache("ETH", "long")
        assert cached is None

    def test_cache_miss_different_side(self, monkeypatch):
        """Long vs short should have separate cache entries."""
        monkeypatch.setattr(
            "app.ai.researcher.get_settings",
            lambda: make_settings(ai_validation_cache_ttl_seconds=900),
        )

        result = ValidationResult(approved=True, confidence=0.7, reason="ok", key_factors=[])
        _store_cache("BTC", "long", result)

        # Short should miss (different cache key)
        cached = _lookup_cache("BTC", "short")
        assert cached is None

    def test_stale_cache_expires(self, monkeypatch):
        """Entry older than TTL should not be returned."""
        monkeypatch.setattr(
            "app.ai.researcher.get_settings",
            lambda: make_settings(ai_validation_cache_ttl_seconds=1),  # 1 second TTL
        )

        result = ValidationResult(approved=True, confidence=0.7, reason="ok", key_factors=[])
        _store_cache("BTC", "long", result)

        # Manually age the entry
        with get_session() as session:
            cache = session.exec(
                select(AIValidationCache).where(AIValidationCache.coin == "BTC")
            ).first()
            if cache:
                cache.researched_at = datetime.now(UTC) - timedelta(seconds=10)
                session.add(cache)
                session.commit()

        cached = _lookup_cache("BTC", "long")
        assert cached is None  # Should be expired

    def test_prune_stale_removes_expired(self, monkeypatch):
        """prune_stale_cache() should delete old entries."""
        monkeypatch.setattr(
            "app.ai.researcher.get_settings",
            lambda: make_settings(ai_validation_cache_ttl_seconds=1),
        )

        result = ValidationResult(approved=True, confidence=0.5, reason="test", key_factors=[])
        _store_cache("BTC", "long", result)

        # Age it
        with get_session() as session:
            cache = session.exec(
                select(AIValidationCache).where(AIValidationCache.coin == "BTC")
            ).first()
            if cache:
                cache.researched_at = datetime.now(UTC) - timedelta(seconds=10)
                session.add(cache)
                session.commit()

        deleted = prune_stale_cache()
        assert deleted >= 1

        # Verify it's gone
        with get_session() as session:
            remaining = session.exec(
                select(AIValidationCache).where(AIValidationCache.coin == "BTC")
            ).all()
            assert len(remaining) == 0


class TestValidationResult:
    """Tests for ValidationResult dataclass."""

    def test_repr(self):
        result = ValidationResult(
            approved=True, confidence=0.8, reason="test", key_factors=["a", "b"]
        )
        r = repr(result)
        assert "approved=True" in r
        assert "0.80" in r

    def test_pending_flag(self):
        result = ValidationResult(
            approved=False, confidence=0.0, reason="pending", key_factors=[], pending=True
        )
        assert result.pending is True
        assert result.approved is False
```

**Step 2: Run tests**

```bash
cd /workspace/repos/hyperliquid-copy-bot
uv run pytest tests/test_ai_validation.py -v
```

Expected: All tests PASS.

---

## Task 10: Run full test suite + lint

**Objective:** Ensure nothing is broken.

**Step 1: Run all tests**

```bash
cd /workspace/repos/hyperliquid-copy-bot
uv run pytest tests/ -v
```

Expected: All existing tests PASS. New AI validation tests PASS.

**Step 2: Run linter**

```bash
cd /workspace/repos/hyperliquid-copy-bot
uv run ruff check app/ai/ app/risk/rules.py app/invo/processor.py app/core/config.py app/db/models.py
```

Expected: No new errors.

---

## Verification Checklist

After all tasks are complete, verify:

- [ ] All AI validation config settings appear in `.env.example`
- [ ] All settings fields exist in `Settings` class with correct defaults
- [ ] `AIValidationCache` table created and queryable
- [ ] `tavily_search()` returns error gracefully when no API key configured
- [ ] `research_coin()` returns `ValidationResult` with correct fields
- [ ] Cache: store → lookup returns same result
- [ ] Cache: different coin misses
- [ ] Cache: stale entry (>TTL) is not returned
- [ ] `prune_stale_cache()` removes expired entries
- [ ] `check_ai_validation()` returns `(True, "")` when `AI_VALIDATION_ENABLED=false`
- [ ] `check_ai_validation()` returns cache hit properly
- [ ] `check_open_rules()` calls `check_ai_validation()` and handles rejection
- [ ] `check_open_rules()` does NOT permanently reject "pending" signals
- [ ] `process_all_signals()` calls `reset_tick_state()` and `prune_stale_cache()`
- [ ] Invo `_process_open()` calls `check_ai_validation()` and handles rejection
- [ ] Telegram alert fires on AI rejection with reason
- [ ] All existing tests still pass
- [ ] No new linter errors
- [ ] `httpx` is in project dependencies (or already available)

---

## Design Decisions

| Decision | Rationale |
|----------|-----------|
| **Tick-based async** | Non-blocking. Multiple coins research in parallel. Pipeline never stalls. |
| **SQLite cache (not vector DB)** | Minimal resource usage. No new services. 1MB storage. Auto-prunes. |
| **Cache per coin+side** | Long and short for same coin may have different contexts. |
| **"pending" signals stay "new"** | Re-evaluated next tick. No signal is lost due to slow research. |
| **Expire on timeout (don't fallback)** | Safer default for paper trading. A missed opportunity < bad trade. |
| **Tavily Search API** | Free tier (1K/mo). AI-optimized results — returns pre-summarized content ideal for LLM consumption. Simpler API than Brave (POST with JSON body, no special headers). |
| **DeepSeek V4 Pro via OpenRouter** | 1M context, tool calling, cheap ($0.002/call). Can swap to Flash for speed. |
| **AI check before exposure rules** | Fast rejection saves computation. Don't calculate stakes for rejected trades. |
| **Per-coin research (not per-signal)** | 5 wallets signal BTC = 1 LLM call. Cache makes the rest instant. |

---

## Resource Impact Summary

| Resource | Before | After | Delta |
|----------|--------|-------|-------|
| RAM | ~500MB (Aria base) | ~510MB | +10MB (httpx sessions + cache) |
| Storage | varies | +1MB/year | Cache auto-prunes |
| CPU | baseline | +0.1% | All heavy work is external |
| Network | baseline | +10KB/call | 1 call per unique coin per 15 min |
| Cost | $0 | ~$2-5/month | ~2,000 LLM calls/month at $0.002 each |
| Dependencies | httpx (already used) | none new | httpx for async HTTP |
| New processes | 0 | 0 | Everything runs in Aria's process |

---

## Future Enhancements (not in this plan)

- **Per-coin cache TTL:** Stable coins 30 min, memecoins 5 min
- **Sentiment scoring:** Track approval rate per source wallet — if wallet X is always rejected by AI, deprioritize it
- **Multi-source research:** Use 2+ search providers for redundancy
- **AI confidence learning:** Track whether AI rejections were "correct" (did price actually move against?) and adjust threshold
- **Model fallback:** If DeepSeek V4 Pro is slow/unavailable, fall back to V4 Flash automatically
