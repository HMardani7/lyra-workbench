# Aria Upgrade #9: Market-Regime-Aware Borderline Gate

**Goal:** Make borderline AI confidence signals (0.50–0.55) pass or reject based on current market conditions — not a flat "always reject."

**Architecture:** New `app/risk/market_regime.py` module that classifies market state from Bitget public tickers, plus a modified `apply_ai_approval_policy()` that upgrades borderline signals in favourable regimes.

**Tech stack:** Python (existing httpx + asyncio), Bitget public REST API (no auth), in-memory TTL cache.

---

## What This Upgrade Does

### The Problem It Solves

Today, any AI validation confidence between 0.50 and 0.55 is immediately rejected — no exceptions. The AI says "this trade looks unobjectionable but I have nothing exciting to say about it," and Aria kills it. This is too blunt:

- A borderline signal during a calm uptrend could easily be a winner.
- A borderline signal during a panic selloff should absolutely be rejected.
- A borderline signal for BTC from a good trader during an orderly dip should probably be copied (small).

### What This Upgrade Changes

Adds a **market regime classifier** that runs before the borderline gate. The classifier fetches 24h change data for ~10 major coins from Bitget's public ticker API and buckets the market into one of four regimes:

| Regime | Condition | Borderline treatment |
|--------|-----------|---------------------|
| **RISK_ON** | BTC +3%+, breadth 60%+ | Upgrade to **weak** — passes with weak-tier checks |
| **NEUTRAL** | BTC −3% to +3% | Upgrade to **weak** only if coin is major **or** trader is strong |
| **PULLBACK** | BTC −3% to −7%, orderly | Upgrade to **weak** only if coin is major **and** no negative trader advisory |
| **PANIC** | BTC < −7%, breadth < 30% | **Reject** — don't catch falling knives |

In all cases the existing weak-tier checks still apply (coin risk, trader health, portfolio crowding, fee edge). An upgrade from borderline → weak does **not** bypass those guards.

### Risk It Prevents

- **Without this:** borderline signals are always killed, even when the market favours them. Missed winners during risk-on.
- **With this:** borderline signals survive when the market says "conditions are favourable enough to take a small bet," but still die in panic.

### Scope

- **Covers:** Invo open signals coming through `_process_open()` and HL signals through `check_open_rules()`.
- **Does NOT cover:** Close signals, increase/decrease signals, stake-size adjustment (future enhancement).
- **Stake-size changes are OUT OF SCOPE** for this upgrade. Borderline→weak trades use standard stake. A follow-up upgrade will add a `regime_stake_multiplier`.

---

## Background

### Current Code Flow

```
Invo webhook → _process_open()
  → get_ai_validation_decision()     # AI researches coin/trader → returns confidence
  → apply_ai_approval_policy()       # deterministic gate on confidence tiers
    ├─ ≥0.65: strong → pass
    ├─ ≥0.60: medium → pass (crowding checks)
    ├─ ≥0.55: weak → pass (coin risk, trader health, crowding)
    ├─ ≥0.50: borderline → REJECT  ← THE PROBLEM
    └─ <0.50: reject
```

### Integration Points

Two callers of `apply_ai_approval_policy()`:

1. **`app/invo/processor.py:170`** — `_process_open()`
2. **`app/risk/rules.py:110`** — `check_open_rules()`

Both pass `(ai_decision, coin, trader, open_trade_count, coin_risk_tier, portfolio_context)`. The new regime logic should be **internal to `apply_ai_approval_policy()`** so neither caller needs to change.

### Existing Market Data

- `BitgetClient.get_ticker(symbol)` exists at `app/bitget/client.py:72` — async, hits `/api/v2/spot/market/tickers?symbol=BTCUSDT`, returns `{symbol, lastPr, change24h, high24h, low24h, ...}`.
- No auth required for public tickers.
- The `change24h` field is a decimal like `"0.0312"` meaning +3.12%.

No batch endpoint is used currently, but `get_ticker` can be called in a loop for ~10 coins (parallel async, <2s total).

### Design Constraints

- **VPS 4GB RAM** — no new services, no ML, no external APIs beyond Bitget public endpoints.
- **Must not block** the webhook path. Use the same event-loop-safe pattern already in `check_ai_validation()` (ThreadPoolExecutor inside running event loops).
- **Must not add latency** — cache the regime result with a 5-min TTL so 99% of calls are in-memory dict lookups.
- **Deterministic + testable** — regime logic is pure math on ticker data, fully unit-testable.

---

## Files to Touch

| File | Action | Purpose |
|------|--------|---------|
| `app/risk/market_regime.py` | **CREATE** | Regime classifier + TTL cache |
| `app/risk/ai_policy.py` | **MODIFY** | Borderline gate: consult regime |
| `app/core/config.py` | **MODIFY** | New config keys |
| `tests/unit/test_market_regime.py` | **CREATE** | Regime classification tests |
| `tests/unit/test_ai_policy.py` | **MODIFY** | Borderline+regime integration tests |

---

## 1. New Module: `app/risk/market_regime.py`

### 1.1 Task: Create market regime classifier

**Objective:** Classify the current market into one of four regimes from Bitget ticker data, with a TTL cache.

**File:** `app/risk/market_regime.py` (new)

**Design:**

```python
from enum import StrEnum

class MarketRegime(StrEnum):
    RISK_ON = "risk_on"       # BTC +3%+, broad strength
    NEUTRAL = "neutral"       # BTC −3% to +3%
    PULLBACK = "pullback"     # BTC −3% to −7%, orderly decline
    PANIC = "panic"           # BTC < −7%, broad weakness
```

**Config-driven watchlist** for breadth calculation — not hardcoded:

```python
# In config.py:
market_regime_enabled: bool = True
market_regime_cache_ttl_seconds: int = 300      # 5 minutes
market_regime_breadth_coins: str = "BTC,ETH,SOL,BNB,XRP,ADA,AVAX,DOT,LINK,SUI"
market_regime_btc_risk_on_pct: float = 3.0       # BTC +3% → risk-on
market_regime_btc_panic_pct: float = -7.0        # BTC −7% → panic
market_regime_breadth_risk_on_pct: float = 60.0  # 60%+ positive → risk-on
market_regime_breadth_panic_pct: float = 30.0    # <30% positive → panic
```

**Classifier function:**

```python
def classify_regime(changes_24h: dict[str, float]) -> MarketRegime:
    """
    changes_24h: {"BTC": 0.0312, "ETH": 0.0185, ...}  # decimal, e.g. 0.03 = +3%
    
    Returns a MarketRegime enum.
    """
    settings = get_settings()
    btc_change = changes_24h.get("BTC", 0.0)
    
    # Breadth: what fraction of coins are positive?
    positives = sum(1 for v in changes_24h.values() if v > 0)
    total = len(changes_24h)
    breadth_pct = (positives / total * 100) if total > 0 else 50.0
    
    risk_on_btc = settings.market_regime_btc_risk_on_pct / 100.0
    panic_btc = settings.market_regime_btc_panic_pct / 100.0
    breadth_risk_on = settings.market_regime_breadth_risk_on_pct
    breadth_panic = settings.market_regime_breadth_panic_pct
    
    if btc_change >= risk_on_btc and breadth_pct >= breadth_risk_on:
        return MarketRegime.RISK_ON
    if btc_change <= panic_btc and breadth_pct < breadth_panic:
        return MarketRegime.PANIC
    if btc_change < -0.03:  # between -3% and -7%
        return MarketRegime.PULLBACK
    return MarketRegime.NEUTRAL
```

**Cached fetcher:**

```python
import time
import asyncio
from dataclasses import dataclass

@dataclass
class _RegimeCache:
    regime: MarketRegime
    fetched_at: float          # monotonic timestamp
    changes: dict[str, float]  # raw changes for debugging

_cache: _RegimeCache | None = None

def get_market_regime() -> MarketRegime:
    """Return the current market regime, fetching if cache is stale."""
    global _cache
    settings = get_settings()
    
    if not getattr(settings, "market_regime_enabled", True):
        return MarketRegime.NEUTRAL  # disabled → always neutral
    
    ttl = getattr(settings, "market_regime_cache_ttl_seconds", 300)
    now = time.monotonic()
    
    if _cache is not None and (now - _cache.fetched_at) < ttl:
        return _cache.regime
    
    # Cache stale or missing — fetch
    changes = _fetch_breadth_changes(settings)
    regime = classify_regime(changes)
    _cache = _RegimeCache(regime=regime, fetched_at=now, changes=changes)
    
    logger.info(
        f"Market regime: {regime.value} | BTC {changes.get('BTC', 0):+.1%} | "
        f"breadth {_breadth_pct(changes):.0f}% positive | "
        f"coins sampled: {len(changes)}"
    )
    return regime


def _fetch_breadth_changes(settings) -> dict[str, float]:
    """Fetch 24h change for all breadth coins from Bitget. Returns {SYMBOL: decimal_change}."""
    coins_raw = getattr(settings, "market_regime_breadth_coins", "BTC,ETH,SOL")
    coins = [c.strip().upper() for c in coins_raw.split(",") if c.strip()]
    
    # Use asyncio to fetch in parallel
    result = asyncio.run(_fetch_all_tickers(coins))
    return result
```

**Event-loop-safe async fetch:**

```python
async def _fetch_all_tickers(coins: list[str]) -> dict[str, float]:
    """Fetch tickers for multiple coins in parallel."""
    from app.bitget.client import BitgetClient
    client = BitgetClient()
    
    async def fetch_one(coin: str) -> tuple[str, float | None]:
        symbol = f"{coin}USDT"
        ticker = await client.get_ticker(symbol)
        if ticker:
            change_str = ticker.get("change24h", "0")
            try:
                change = float(change_str)
                return (coin, change)
            except (ValueError, TypeError):
                return (coin, None)
        return (coin, None)
    
    tasks = [fetch_one(c) for c in coins]
    results = await asyncio.gather(*tasks, return_exceptions=True)
    
    changes: dict[str, float] = {}
    for r in results:
        if isinstance(r, Exception):
            continue
        coin, change = r
        if change is not None:
            changes[coin] = change
    
    # Always ensure BTC is present (fallback to 0.0)
    if "BTC" not in changes:
        logger.warning("BTC ticker fetch failed — regime defaults to neutral")
        changes["BTC"] = 0.0
    
    return changes
```

**⚠️ Event loop safety:** `_fetch_breadth_changes()` calls `asyncio.run()`. When called from the Invo webhook (inside FastAPI's event loop), this will fail with "cannot be called from a running event loop." Use the same guard pattern already deployed in `check_ai_validation()`:

```python
def _fetch_breadth_changes(settings) -> dict[str, float]:
    coins_raw = getattr(settings, "market_regime_breadth_coins", "BTC,ETH,SOL")
    coins = [c.strip().upper() for c in coins_raw.split(",") if c.strip()]
    
    try:
        asyncio.get_running_loop()
        inside_loop = True
    except RuntimeError:
        inside_loop = False
    
    if not inside_loop:
        return asyncio.run(_fetch_all_tickers(coins))
    
    import concurrent.futures
    with concurrent.futures.ThreadPoolExecutor() as pool:
        return pool.submit(
            asyncio.run, _fetch_all_tickers(coins)
        ).result(timeout=15)
```

### 1.2 Verification

```bash
# 1. Import check
cd /srv/hermes-repos/hyperliquid-copy-bot
uv run python -c "
from app.risk.market_regime import MarketRegime, classify_regime
print('Import OK')
print('Regimes:', list(MarketRegime))
"

# 2. Classification logic test (no API calls)
uv run python -c "
from app.risk.market_regime import classify_regime
# Simulate risk-on: BTC +5%, 8/10 positive
changes = {'BTC': 0.05, 'ETH': 0.03, 'SOL': 0.04, 'BNB': 0.02, 'XRP': 0.01, 'ADA': -0.01, 'AVAX': 0.03, 'DOT': 0.01, 'LINK': 0.02, 'SUI': 0.06}
print('Risk-on:', classify_regime(changes))

# Simulate panic: BTC -8%, 2/10 positive
changes = {'BTC': -0.08, 'ETH': -0.09, 'SOL': -0.11, 'BNB': -0.07, 'XRP': -0.05, 'ADA': -0.10, 'AVAX': -0.12, 'DOT': 0.01, 'LINK': -0.06, 'SUI': 0.02}
print('Panic:', classify_regime(changes))

# Simulate neutral: BTC -1%
changes = {'BTC': -0.01, 'ETH': -0.02, 'SOL': 0.01, 'BNB': -0.01, 'XRP': 0.00, 'ADA': 0.02, 'AVAX': -0.03, 'DOT': 0.01, 'LINK': -0.01, 'SUI': -0.02}
print('Neutral:', classify_regime(changes))

# Simulate pullback: BTC -5%
changes = {'BTC': -0.05, 'ETH': -0.04, 'SOL': -0.06, 'BNB': -0.03, 'XRP': 0.01, 'ADA': -0.04, 'AVAX': -0.05, 'DOT': -0.03, 'LINK': 0.01, 'SUI': -0.02}
print('Pullback:', classify_regime(changes))
"

# 3. Full integration: fetch real tickers + classify
uv run python -c "
from app.risk.market_regime import get_market_regime
regime = get_market_regime()
print(f'Current regime: {regime}')
"
```

---

## 2. Modified Module: `app/risk/ai_policy.py`

### 2.1 Task: Add regime-aware borderline handling

**Objective:** Replace the flat "borderline → reject" with a regime-gated decision.

**File:** `app/risk/ai_policy.py` (modify lines 131–135)

**Current code (to replace):**

```python
    if confidence >= borderline:
        reason = f"AI borderline approval ({confidence:.0%})"
        if action == "observe":
            return False, f"{reason}: observe-only; not opening trade{_fee_hurdle_text(confidence)}"
        return False, f"{reason} rejected by borderline gate{_fee_hurdle_text(confidence)}"
```

**New code:**

```python
    if confidence >= borderline:
        from app.risk.market_regime import MarketRegime, get_market_regime
        
        regime = get_market_regime()
        reason = f"AI borderline approval ({confidence:.0%})"
        
        # Step 1: regime decides whether borderline can be upgraded
        regime_allow, regime_label = _regime_borderline_policy(
            regime, coin, normalized_coin_tier, trader, same_trader_open_count,
            effective_open_trade_count, crowded_at, max_trader_open,
            resolved_trader_status
        )
        
        if not regime_allow:
            return False, (
                f"{reason} rejected: {regime_label} "
                f"(regime={regime.value}){_fee_hurdle_text(confidence)}"
            )
        
        # Step 2: upgrade to weak — reapply weak-tier checks
        weak_reason = f"{reason} upgraded to weak ({regime_label}, regime={regime.value})"
        return _apply_weak_checks(
            confidence, coin, normalized_coin_tier, trader,
            same_trader_open_count, effective_open_trade_count,
            crowded_at, max_trader_open, resolved_trader_status,
            weak_reason
        )
```

**New helper — `_regime_borderline_policy()`:**

```python
def _regime_borderline_policy(
    regime: "MarketRegime",
    coin: str,
    normalized_coin_tier: str,
    trader: str,
    same_trader_open_count: int,
    effective_open_trade_count: int,
    crowded_at: int,
    max_trader_open: int,
    trader_status: str | None,
) -> tuple[bool, str]:
    """Determine whether a borderline signal can be upgraded to weak based on market regime.
    
    Returns (allowed, label_for_reason_string).
    """
    from app.risk.market_regime import MarketRegime
    
    if regime == MarketRegime.PANIC:
        return False, "panic regime — broad selloff, not buying dips"
    
    if regime == MarketRegime.RISK_ON:
        return True, "risk-on regime — market tailwinds"
    
    if regime == MarketRegime.PULLBACK:
        # Only major coins or strong traders in a pullback
        if normalized_coin_tier == "major":
            return True, "pullback regime — major coin dip-buy"
        trader_ok = trader_status and trader_status in ("strong", "neutral")
        if trader_ok:
            return True, f"pullback regime — trusted trader ({trader})"
        return False, (
            f"pullback regime — {coin} is not major and trader {trader} "
            f"is {trader_status or 'unknown'}"
        )
    
    # NEUTRAL regime
    if normalized_coin_tier == "major":
        return True, "neutral regime — major coin"
    trader_ok = trader_status and trader_status in ("strong", "neutral")
    if trader_ok:
        return True, f"neutral regime — trusted trader ({trader})"
    return False, (
        f"neutral regime — {coin} is not major and trader {trader} "
        f"is {trader_status or 'unknown'}"
    )
```

**⚠️ Important:** The `_apply_weak_checks()` function should be extracted from the existing weak-tier block (lines 93–129). Currently the weak checks are inline. Extract them into a helper so borderline-upgraded signals go through the same checks:

```python
def _apply_weak_checks(
    confidence: float,
    coin: str,
    normalized_coin_tier: str,
    trader: str,
    same_trader_open_count: int,
    effective_open_trade_count: int,
    crowded_at: int,
    max_trader_open: int,
    trader_status: str | None,
    prefix_reason: str,
) -> tuple[bool, str]:
    """Apply weak-tier checks: coin risk, trader health, crowding. Returns (allowed, reason)."""
    if normalized_coin_tier == "high-risk":
        return (
            False,
            f"{prefix_reason} rejected: {coin} is high-risk"
            f"{_fee_hurdle_text(confidence)}",
        )
    
    resolved_trader_status = trader_status
    if resolved_trader_status is None and _setting("trader_advisory_enabled", True):
        resolved_trader_status = classify_trader_by_name(trader).status
    resolved_trader_status = resolved_trader_status or "neutral"
    normalized_trader_status = resolved_trader_status.lower().replace("_", "-")
    
    if normalized_trader_status in {"caution", "unfollow-candidate"}:
        return (
            False,
            f"{prefix_reason} rejected: trader {trader} is "
            f"{resolved_trader_status}{_fee_hurdle_text(confidence)}",
        )
    if same_trader_open_count >= max_trader_open:
        return (
            False,
            f"{prefix_reason} rejected: same trader {trader} "
            f"already has {same_trader_open_count}/{max_trader_open} open trades"
            f"{_fee_hurdle_text(confidence)}",
        )
    if effective_open_trade_count >= crowded_at:
        return (
            False,
            f"{prefix_reason} rejected: portfolio crowded "
            f"with {effective_open_trade_count} open trades{_fee_hurdle_text(confidence)}",
        )
    return True, f"{prefix_reason}: {_fee_hurdle_text(confidence)}"
```

**Then replace the existing weak block (lines 93–129) with a call to `_apply_weak_checks()`:**

```python
    if confidence >= weak:
        return _apply_weak_checks(
            confidence, coin, normalized_coin_tier, trader,
            same_trader_open_count, effective_open_trade_count,
            crowded_at, max_trader_open, resolved_trader_status,
            f"AI weak approval ({confidence:.0%})"
        )
```

**⚠️ `resolved_trader_status` initialization:** Currently the weak block computes `resolved_trader_status` internally. After extraction, `resolved_trader_status` must be computed before the weak-tier `if` block so it's available for both weak and borderline-upgraded paths. Move the computation up, right after `normalized_coin_tier` is set (around line 73):

```python
    resolved_trader_status = trader_status
    if resolved_trader_status is None and _setting("trader_advisory_enabled", True):
        resolved_trader_status = classify_trader_by_name(trader).status
    resolved_trader_status = resolved_trader_status or "neutral"
```

Then remove the duplicate computation from `_apply_weak_checks()` (the helper should use what's passed in).

Wait — actually cleaner: `_apply_weak_checks()` can accept `trader_status` as already-resolved (non-None). The caller resolves it once. That way the helper is pure — no side effects, no config reads for trader status.

Let me revise: the caller resolves `trader_status`, passes it to `_apply_weak_checks()` as `resolved_trader_status`. In the borderline path, we've already computed it. In the weak path, we compute it just before calling. This avoids double-computation.

**Final modified `apply_ai_approval_policy()` flow:**

```python
def apply_ai_approval_policy(ai, *, coin, trader, open_trade_count,
                              coin_risk_tier="standard", trader_status=None,
                              portfolio_context=None):
    if not ai.allowed:
        return False, ai.reason
    
    if not _setting("ai_borderline_gate_enabled", True):
        return ai.allowed, ai.reason
    
    confidence = ai.confidence
    if confidence is None:
        return ai.allowed, ai.reason
    
    # ... thresholds, crowded_at, max_trader_open, portfolio_context ...
    
    # Resolve trader status once, up front (moved from inside weak block)
    resolved_trader_status = trader_status
    if resolved_trader_status is None and _setting("trader_advisory_enabled", True):
        resolved_trader_status = classify_trader_by_name(trader).status
    resolved_trader_status = resolved_trader_status or "neutral"
    
    if confidence >= strong:
        return True, f"AI strong approval ({confidence:.0%}): {ai.reason}"
    
    if confidence >= medium:
        # ... existing medium checks (unchanged) ...
    
    if confidence >= weak:
        return _apply_weak_checks(
            confidence, coin, normalized_coin_tier, trader,
            same_trader_open_count, effective_open_trade_count,
            crowded_at, max_trader_open, resolved_trader_status,
            f"AI weak approval ({confidence:.0%})"
        )
    
    if confidence >= borderline:
        # NEW: regime-aware borderline handling
        from app.risk.market_regime import get_market_regime
        regime = get_market_regime()
        reason = f"AI borderline approval ({confidence:.0%})"
        
        regime_allow, regime_label = _regime_borderline_policy(
            regime, coin, normalized_coin_tier, trader,
            same_trader_open_count, effective_open_trade_count,
            crowded_at, max_trader_open, resolved_trader_status
        )
        
        if not regime_allow:
            return False, (
                f"{reason} rejected: {regime_label} "
                f"(regime={regime.value}){_fee_hurdle_text(confidence)}"
            )
        
        return _apply_weak_checks(
            confidence, coin, normalized_coin_tier, trader,
            same_trader_open_count, effective_open_trade_count,
            crowded_at, max_trader_open, resolved_trader_status,
            f"{reason} upgraded to weak ({regime_label}, regime={regime.value})"
        )
    
    return False, f"AI confidence too low ({confidence:.0%} < {borderline:.0%}): {ai.reason}"
```

### 2.2 Verification

```bash
# 1. Import check after modifications
cd /srv/hermes-repos/hyperliquid-copy-bot
uv run python -c "
from app.risk.ai_policy import apply_ai_approval_policy, _regime_borderline_policy
print('Import OK')
"

# 2. Unit test: borderline + risk_on → should pass (upgraded to weak)
uv run pytest tests/unit/test_ai_policy.py -xvs -k "borderline"

# 3. Integration: run the full policy with mock regime data
uv run python -c "
from app.risk.ai_policy import apply_ai_approval_policy
from app.ai.validator import AIValidationDecision
from app.risk.market_regime import _cache, _RegimeCache, MarketRegime
import time

# Inject a fake cached regime (risk-on)
_cache = _RegimeCache(
    regime=MarketRegime.RISK_ON,
    fetched_at=time.monotonic(),
    changes={'BTC': 0.05, 'ETH': 0.03}
)

ai = AIValidationDecision(allowed=True, confidence=0.52, reason='no negative news')
allowed, reason = apply_ai_approval_policy(
    ai, coin='BTC', trader='goodtrader', open_trade_count=2,
    coin_risk_tier='major'
)
print(f'Risk-on borderline BTC: allowed={allowed}, reason={reason}')
assert allowed, 'Should pass in risk-on!'
print('PASS')

# Now test panic: borderline → reject
_cache = _RegimeCache(
    regime=MarketRegime.PANIC,
    fetched_at=time.monotonic(),
    changes={'BTC': -0.09, 'ETH': -0.10}
)
allowed, reason = apply_ai_approval_policy(
    ai, coin='BTC', trader='goodtrader', open_trade_count=2,
    coin_risk_tier='major'
)
print(f'Panic borderline BTC: allowed={allowed}, reason={reason}')
assert not allowed, 'Should reject in panic!'
print('PASS')
"
```

---

## 3. New Config Keys: `app/core/config.py`

### 3.1 Task: Add market regime config

**Objective:** Add configurable thresholds so the regime classifier can be tuned without code changes.

**File:** `app/core/config.py` (add to `Settings` class, after the borderline gate block around line 118)

**New keys:**

```python
    # Market Regime Classification
    market_regime_enabled: bool = True
    market_regime_cache_ttl_seconds: int = 300  # 5 min — how often to re-fetch tickers
    market_regime_breadth_coins: str = "BTC,ETH,SOL,BNB,XRP,ADA,AVAX,DOT,LINK,SUI"
    market_regime_btc_risk_on_pct: float = 3.0           # BTC +3%+ → risk-on
    market_regime_btc_panic_pct: float = -7.0            # BTC −7%+ → panic
    market_regime_breadth_risk_on_pct: float = 60.0       # 60%+ coins positive → risk-on
    market_regime_breadth_panic_pct: float = 30.0         # <30% positive → panic
```

### 3.2 Verification

```bash
uv run python -c "
from app.core.config import get_settings
s = get_settings()
print('market_regime_enabled:', s.market_regime_enabled)
print('breadth_coins:', s.market_regime_breadth_coins)
print('btc_risk_on_pct:', s.market_regime_btc_risk_on_pct)
"
```

---

## 4. Tests: `tests/unit/test_market_regime.py`

### 4.1 Task: Unit tests for regime classification

**Objective:** Cover all regime boundaries, edge cases, and the borderline policy integration.

**File:** `tests/unit/test_market_regime.py` (new)

**Test cases:**

| Test | Input | Expected regime |
|------|-------|----------------|
| `test_risk_on` | BTC +5%, 8/10 positive | RISK_ON |
| `test_risk_on_boundary` | BTC +3.0%, 60% positive | RISK_ON |
| `test_risk_on_just_below_btc` | BTC +2.9%, 70% positive | NEUTRAL (BTC below threshold) |
| `test_neutral_flat` | BTC +1%, 5/10 positive | NEUTRAL |
| `test_neutral_slightly_down` | BTC -2%, 4/10 positive | NEUTRAL |
| `test_pullback` | BTC -5%, 3/10 positive | PULLBACK |
| `test_pullback_upper_boundary` | BTC -3.1%, 5/10 positive | PULLBACK |
| `test_panic` | BTC -8%, 2/10 positive | PANIC |
| `test_panic_boundary` | BTC -7.0%, 29% positive | PANIC |
| `test_panic_but_good_breadth` | BTC -8%, 40% positive | PULLBACK (breadth saves from panic) |
| `test_empty_changes` | {} | NEUTRAL (BTC defaults to 0.0) |
| `test_only_btc_positive` | BTC +5%, all others 0 | NEUTRAL (breadth 10%) |
| `test_disabled` | market_regime_enabled=False | NEUTRAL (always) |

**Additional test file: `tests/unit/test_ai_policy.py` additions:**

| Test | Input | Expected |
|------|-------|----------|
| `test_borderline_risk_on_major_coin_passes` | borderline + RISK_ON + BTC | allowed=True |
| `test_borderline_risk_on_meme_coin_passes` | borderline + RISK_ON + PENGU, neutral trader | allowed=True (risk-on upgrades regardless) |
| `test_borderline_panic_rejects` | borderline + PANIC + BTC | allowed=False |
| `test_borderline_pullback_major_passes` | borderline + PULLBACK + BTC, trader=good | allowed=True |
| `test_borderline_pullback_meme_rejects` | borderline + PULLBACK + PENGU, trader=neutral | allowed=False |
| `test_borderline_neutral_major_passes` | borderline + NEUTRAL + SOL, trader=unknown | allowed=True |
| `test_borderline_neutral_bad_trader_rejects` | borderline + NEUTRAL + standard coin, trader=caution | allowed=False |
| `test_borderline_upgraded_still_checks_high_risk` | borderline + RISK_ON + PENGU (high-risk) + caution trader | allowed=False (weak checks: high-risk coin) |

---

## 5. Verification Checklist

After implementation on the VPS:

- [ ] `uv run python -c "from app.risk.market_regime import MarketRegime, get_market_regime; print(get_market_regime())"` — returns a real regime from live Bitget data
- [ ] `uv run python -c "from app.core.config import get_settings; s=get_settings(); assert s.market_regime_enabled"` — config loaded
- [ ] `uv run pytest tests/unit/test_market_regime.py -xvs` — all regime tests pass
- [ ] `uv run pytest tests/unit/test_ai_policy.py -xvs -k "borderline"` — all borderline+regime tests pass
- [ ] `uv run pytest tests/unit/ -x` — full unit suite green
- [ ] `docker logs hyperliquid-copy-bot --tail 100 2>&1 | grep -i "market regime"` — regime fetch logged on startup
- [ ] `docker logs hyperliquid-copy-bot --tail 200 2>&1 | grep -i "upgraded to weak"` — after a real borderline signal passes, reason string contains "upgraded to weak" and "regime="
- [ ] No new error traces in container logs after restart
- [ ] Invo signals with borderline confidence in a favourable regime produce "copied" decisions (verified via `aria-db-query`)

---

## 6. Design Decisions

| Decision | Rationale |
|----------|-----------|
| Regime classifier fetches Bitget tickers, not CoinGecko/CMC | No new API keys or rate limits. Bitget public API is free and already integrated. |
| Cache TTL 5 minutes | Market regime changes slowly. 5 min avoids API spam on every signal while staying current. |
| Regime is internal to `apply_ai_approval_policy()` | No caller signature changes. The policy module owns the decision. |
| `_apply_weak_checks()` extracted as a helper | DRY. Borderline-upgraded signals must go through the same checks as naturally-weak ones. |
| `resolved_trader_status` computed once, up front | Avoids double DB lookup. Available to both weak and borderline paths. |
| Stake multiplier left for follow-up | Keeps scope tight. This upgrade is about the allow/reject decision only. |
| Breadth uses a configurable coin list, not "top-N by market cap" | Simple, testable, no extra API calls to rank coins. User can tune the list. |
| Disabled regime → NEUTRAL | Safe default. No unexpected behaviour when the feature is off. |
| Regime fetch failures → NEUTRAL with BTC=0.0 | Graceful degradation. A Bitget API hiccup shouldn't kill all borderline signals. |

---

## 7. Future Enhancements

- **Stake multiplier by regime:** RISK_ON = 1.0x, PULLBACK = 0.5x, NEUTRAL = 0.75x. Smaller bets when less certain.
- **Regime-based adjustment for medium/strong tiers:** In PANIC, even medium confidence could get a stake haircut.
- **Fear & Greed Index integration:** As a secondary regime signal alongside BTC/breadth data.
- **Regime change Telegram alerts:** Notify when regime flips (e.g., "Market regime changed: NEUTRAL → PANIC").
- **Regime history in DB:** Store regime changes for backtesting correlation with trade outcomes.

---

## 8. Resource Impact

| Resource | Impact |
|----------|--------|
| **RAM** | Negligible (~1KB for cached regime dict) |
| **CPU** | ~10 async HTTP calls every 5 minutes (<2s wall time) |
| **API calls** | 10 Bitget public ticker calls per 5 min = ~2,880/day (free, no rate limit issues) |
| **Disk** | None (no DB writes for regime in this upgrade) |
