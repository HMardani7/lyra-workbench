# Defensive Trade Triage Implementation Plan

> **For Hermes:** Use subagent-driven-development skill to implement this plan task-by-task.

**Goal:** Stop Aria from copying the highest-risk Invo signals by enforcing trader and leverage hard gates, reducing exposure, raising AI threshold, and fixing broken AI feedback PnL recording.

**Architecture:** Add deterministic risk gates before AI confidence is allowed to approve a trade. Runtime config overrides protect the live VPS from stale Docker env vars. The Invo Reader filter should be tightened too, but Aria remains the final source of truth and rejects unsafe signals even if Invo forwards them.

**Tech Stack:** Python 3.11+, SQLModel, Pydantic Settings, SQLite, pytest, Docker, Node.js Invo Reader env config.

---

## What This Upgrade Does

### The Problem It Solves

Aria's current live data shows the bad PnL is concentrated in two buckets:

- High source leverage signals:
  - `>20x`: 6 closed trades, `-$22.96`
  - `11–20x`: 4 closed trades, `-$20.98`
  - `6–10x`: 10 closed trades, `+$6.76`
- Underperforming traders:
  - `kzs`: 4 closed since reset, 1W/3L, `-$15.05`
  - `beef131412`: 2 closed since reset, `-$6.79`
  - `luxora`: all-time 20% WR, `-$32.69`; AI is now rejecting many luxora signals but existing open luxora trades are red

**Root cause:** Aria treats the LLM as the main quality filter. The LLM can approve signals at 0.50–0.72 confidence even when the source trader or source leverage profile is structurally bad. Current Docker env also allows `INVO_MAX_SOURCE_LEVERAGE=40`, `MAX_TOTAL_EXPOSURE_PCT=0.80`, and `MAX_OPEN_TRADES=20`.

### What This Upgrade Changes

Aria becomes defensive by default:

- If source leverage is above `10x` → **ignore** before AI or execution
- If trader is not in the temporary allowlist → **ignore** before AI or execution
- If AI confidence is below `0.60` → **ignore**
- If total exposure would exceed `50%` → **ignore**
- If max open trades is already `8` → **ignore**
- If an AI-approved trade closes → feedback records the **actual** PnL, not stale zero PnL

### Risk It Prevents

- **High-leverage signal copying:** A 20–40x Invo trade often represents a fragile scalp. Aria's spot-simulated copy can inherit terrible entry timing without the source trader's risk management.
- **Bad trader drift:** A trader can look acceptable after a small sample, then quickly become the largest loss driver. Hard allowlists let Hammad decide who Aria can follow while stats mature.
- **Exposure pile-up:** 20 open trades / 80% exposure lets a bad regime compound quickly.
- **Broken learning loop:** AI feedback currently stores `trade_pnl=0.0` because `record_outcome()` receives a stale pre-close trade object. Auto-tune cannot learn from zero-PnL rows.

### Configurability

| Setting | Defensive Default | What It Does |
|---------|-------------------|--------------|
| `INVO_ALLOWED_TRADERS` | `vortex_legion,bustersquad,blocked` | Temporary allowlist; blocks `luxora`, `kzs`, `beef131412` |
| `INVO_MAX_SOURCE_LEVERAGE` | `10` | Rejects >10x source leverage inside Aria |
| `MAX_SOURCE_LEVERAGE_TO_FORWARD` | `10` | Invo Reader stops forwarding >10x to reduce noise |
| `AI_VALIDATION_MIN_CONFIDENCE` | `0.60` | Requires stronger AI approval |
| `MAX_TOTAL_EXPOSURE_PCT` | `0.50` | Caps total open exposure to 50% of equity |
| `MAX_OPEN_TRADES` | `8` | Prevents broad basket of weak entries |
| `MAX_OPEN_TRADES_PER_WALLET` | `3` | Prevents one trader/source from dominating |
| `MAX_STAKE_PCT` | `0.03` | Caps per-trade notional lower than current 5% effective cap |
| `DEFAULT_STAKE_PCT` | `0.015` | Lowers base trade size while testing |
| `STOP_LOSS_PCT` | `-0.08` | Emergency stop; not expected to save entries, but limits deep bleed |

### Scope

- Applies to **new Invo open signals only**.
- Does **not** force-close current open trades.
- Does **not** change live Bitget trading; Aria is still paper mode.
- Does **not** remove traders from Invo itself; it only changes what Aria accepts/copies.
- Does **not** redesign AI prompts. This is deterministic triage before AI.

---

## Current Production Facts to Preserve

- Authoritative Aria repo: `/srv/hermes-repos/hyperliquid-copy-bot/`
- Running Aria container: `hyperliquid-copy-bot`
- Running Invo Reader container: `invo-reader-xvfb`
- Live Aria DB in container: `/app/data/hyperliquid_copy_bot.db`
- Use `workdir=/home/hermes` for VPS terminal calls.
- Use `uv run python`, not system `python`, inside the Aria container.
- Do not print `.env` secrets.

---

## Files to Touch

| File | Action | Purpose |
|------|--------|---------|
| `app/core/config.py` | Modify | Defensive runtime overrides and safer defaults |
| `app/invo/processor.py` | Modify | Fix AI feedback recording to use refreshed closed trade object |
| `tests/unit/test_config.py` | Modify/Add tests | Verify defensive runtime defaults |
| `tests/unit/test_invo_processor_feedback.py` | Create | Verify feedback receives actual closed trade PnL |
| `.env.example` | Modify | Document defensive config values |
| Invo Reader host `.env` | Modify on VPS only | Set `MAX_SOURCE_LEVERAGE_TO_FORWARD=10` |

---

## Implementation Tasks

### Task 1: Add defensive runtime overrides in Aria config

**Objective:** Force safe live defaults even when root-owned Docker env vars are stale.

**Files:**
- Modify: `app/core/config.py`
- Modify: `tests/unit/test_config.py`

**Step 1: Add or update test coverage**

Append tests like this to `tests/unit/test_config.py`:

```python
def test_defensive_runtime_overrides_are_applied(monkeypatch):
    monkeypatch.setenv("INVO_ALLOWED_TRADERS", "*")
    monkeypatch.setenv("INVO_MAX_SOURCE_LEVERAGE", "40")
    monkeypatch.setenv("MAX_TOTAL_EXPOSURE_PCT", "0.80")
    monkeypatch.setenv("MAX_OPEN_TRADES", "20")
    monkeypatch.setenv("AI_VALIDATION_MIN_CONFIDENCE", "0.4")

    from app.core.config import Settings

    settings = Settings()

    assert settings.invo_allowed_traders == "vortex_legion,bustersquad,blocked"
    assert settings.invo_max_source_leverage == 10.0
    assert settings.ai_validation_min_confidence == 0.60
    assert settings.max_total_exposure_pct == 0.50
    assert settings.max_open_trades == 8
    assert settings.max_open_trades_per_wallet == 3
    assert settings.default_stake_pct == 0.015
    assert settings.max_stake_pct == 0.03
    assert settings.stop_loss_pct == -0.08
```

**Step 2: Run the failing test**

```bash
cd /srv/hermes-repos/hyperliquid-copy-bot
uv run pytest tests/unit/test_config.py::test_defensive_runtime_overrides_are_applied -q
```

Expected: FAIL until config override is changed.

**Step 3: Update `Settings.__init__`**

In `app/core/config.py`, update the runtime override block:

```python
    def __init__(self, **kwargs):
        """Runtime defaults for VPS deployment.

        The VPS container has older env vars baked into docker-compose. These
        overrides keep the live service aligned with defensive paper-trading
        defaults without requiring a root-owned .env edit/recreate. API keys still
        come from the environment.
        """
        super().__init__(**kwargs)

        # Defensive trade triage — deterministic risk gates before AI approval.
        self.invo_allowed_traders = "vortex_legion,bustersquad,blocked"
        self.invo_max_source_leverage = 10.0
        self.default_stake_pct = 0.015
        self.max_stake_pct = 0.03
        self.max_total_exposure_pct = 0.50
        self.max_open_trades = 8
        self.max_open_trades_per_wallet = 3
        self.max_exposure_per_wallet_pct = 0.25
        self.max_exposure_per_coin_pct = 0.15
        self.reserve_cash_pct = 0.20
        self.stop_loss_pct = -0.08

        # AI quality threshold — still a secondary filter after hard risk gates.
        self.ai_validation_min_confidence = 0.60
        self.ai_validation_fallback_on_timeout = False
        self.ai_validation_blocking_timeout_seconds = 45
        self.ai_validation_model = "deepseek/deepseek-v4-flash"
        self.ai_auto_tune_enabled = True
        self.ai_auto_tune_target_win_rate = 0.50
```

**Step 4: Run test again**

```bash
uv run pytest tests/unit/test_config.py::test_defensive_runtime_overrides_are_applied -q
```

Expected: PASS.

**Step 5: Commit**

```bash
git add app/core/config.py tests/unit/test_config.py
git commit -m "chore: add defensive Aria runtime defaults"
```

---

### Task 2: Verify existing Invo processor gates use the config

**Objective:** Confirm `_process_open()` already applies `invo_allowed_traders` and `invo_max_source_leverage` before execution.

**Files:**
- Inspect: `app/invo/processor.py`
- Modify tests only if coverage is missing.

**Step 1: Locate checks**

```bash
grep -n "invo_allowed_traders\|invo_max_source_leverage\|source_leverage" app/invo/processor.py
```

Expected: `_process_open()` rejects disallowed traders and source leverage above max before executing a trade.

**Step 2: Add tests if missing**

If there is no existing Invo processor test for these two paths, create `tests/unit/test_invo_processor_risk_gates.py` with targeted unit tests that monkeypatch execution and assert no trade opens when:

- `payload.trader = "kzs"` while allowlist is `vortex_legion,bustersquad,blocked`
- `payload.source_leverage = 20.0` while `invo_max_source_leverage = 10.0`

Use existing test fixtures/patterns from nearby tests. Keep tests small; mock DB/session if needed.

**Step 3: Run targeted tests**

```bash
uv run pytest tests/unit/test_invo_processor_risk_gates.py -q
```

Expected: PASS.

**Step 4: Commit if tests were added**

```bash
git add tests/unit/test_invo_processor_risk_gates.py
git commit -m "test: cover Invo defensive risk gates"
```

---

### Task 3: Fix AI feedback to record actual closed trade PnL

**Objective:** Ensure `ai_validation_outcomes.trade_pnl` stores real net PnL, not `0.0` from the stale pre-close trade object.

**Files:**
- Modify: `app/invo/processor.py:464-524`
- Create: `tests/unit/test_invo_processor_feedback.py`

**Bug:** `_execute_close()` updates `db_trade.pnl_amount`, commits it, then calls `record_outcome(trade)`. `trade` is the stale object originally passed into the function; it does not have the new PnL fields.

**Step 1: Write failing test**

Create `tests/unit/test_invo_processor_feedback.py` with a focused monkeypatch test. If importing full `processor.py` requires app dependencies, follow existing test patterns. Core expectation:

```python
def test_execute_close_records_feedback_with_actual_net_pnl(monkeypatch):
    """Regression: AI feedback must receive the refreshed closed DB trade."""
    # Build a fake trade with ai_approved=True and an entry price.
    # Monkeypatch app.ai.feedback.record_outcome to capture the passed trade.
    # Execute close with exit_price below entry.
    # Assert captured_trade.pnl_amount is negative, not None/0.
```

If a direct `_execute_close()` unit test is too expensive because of DB/session coupling, use a live temporary SQLite test DB fixture from existing paper engine tests.

**Step 2: Run failing test**

```bash
uv run pytest tests/unit/test_invo_processor_feedback.py -q
```

Expected: FAIL because `record_outcome()` sees the stale object.

**Step 3: Patch `_execute_close()`**

Change the final feedback block from:

```python
    record_outcome(trade)
```

to a refreshed DB object pattern:

```python
    with get_session() as session:
        closed_trade = session.exec(select(PaperTrade).where(PaperTrade.id == trade.id)).first()

    if closed_trade:
        record_outcome(closed_trade)
    else:
        logger.warning(f"[INVO] Could not reload closed trade {trade.id} for AI feedback")
```

Important: `PaperTrade` is already imported inside `_execute_close()`, and `select`/`get_session` are available in the module/function context. If not, add imports explicitly.

**Step 4: Run test again**

```bash
uv run pytest tests/unit/test_invo_processor_feedback.py -q
```

Expected: PASS.

**Step 5: Run related tests**

```bash
uv run pytest tests/unit/test_pnl.py tests/unit/test_config.py tests/unit/test_invo_processor_feedback.py -q
```

Expected: PASS.

**Step 6: Commit**

```bash
git add app/invo/processor.py tests/unit/test_invo_processor_feedback.py
git commit -m "fix: record AI feedback with closed trade pnl"
```

---

### Task 4: Update `.env.example` with defensive defaults

**Objective:** Make new deployments match the defensive live policy.

**Files:**
- Modify: `.env.example`

**Step 1: Update or add values**

Ensure `.env.example` includes:

```bash
# Defensive Invo triage
INVO_ALLOWED_TRADERS=vortex_legion,bustersquad,blocked
INVO_MAX_SOURCE_LEVERAGE=10
MAX_COPY_LEVERAGE=8

# Defensive exposure / sizing
DEFAULT_STAKE_PCT=0.015
MAX_STAKE_PCT=0.03
MAX_TOTAL_EXPOSURE_PCT=0.50
MAX_OPEN_TRADES=8
MAX_OPEN_TRADES_PER_WALLET=3
MAX_EXPOSURE_PER_WALLET_PCT=0.25
MAX_EXPOSURE_PER_COIN_PCT=0.15
RESERVE_CASH_PCT=0.20

# AI validation
AI_VALIDATION_MIN_CONFIDENCE=0.60
AI_VALIDATION_FALLBACK_ON_TIMEOUT=false

# Emergency loss cap
STOP_LOSS_PCT=-0.08
```

**Step 2: Verify no secrets**

```bash
git diff -- .env.example
```

Expected: only example/default values, no real API keys or tokens.

**Step 3: Commit**

```bash
git add .env.example
git commit -m "docs: document defensive Aria defaults"
```

---

### Task 5: Deploy Aria code to the running container

**Objective:** Apply Aria-side defensive gates and feedback fix to the live paper bot.

**Files:**
- Source repo: `/srv/hermes-repos/hyperliquid-copy-bot/`
- Container: `hyperliquid-copy-bot`

**Step 1: Verify source repo is clean except intended commits**

```bash
cd /srv/hermes-repos/hyperliquid-copy-bot
git status --short --branch
```

Expected: clean working tree on `main` after commits.

**Step 2: Copy modified files into container**

```bash
docker cp app/core/config.py hyperliquid-copy-bot:/app/app/core/config.py
docker cp app/invo/processor.py hyperliquid-copy-bot:/app/app/invo/processor.py
```

**Step 3: Verify imports/config inside container**

```bash
docker exec hyperliquid-copy-bot uv run python - <<'PY'
from app.core.config import get_settings
from app.invo.processor import _process_open
s = get_settings()
print('allowed:', s.invo_allowed_traders)
print('max_source_lev:', s.invo_max_source_leverage)
print('ai_min_conf:', s.ai_validation_min_confidence)
print('max_total_exposure:', s.max_total_exposure_pct)
print('max_open_trades:', s.max_open_trades)
print('default_stake_pct:', s.default_stake_pct)
print('max_stake_pct:', s.max_stake_pct)
print('stop_loss_pct:', s.stop_loss_pct)
PY
```

Expected values:

```text
allowed: vortex_legion,bustersquad,blocked
max_source_lev: 10.0
ai_min_conf: 0.6
max_total_exposure: 0.5
max_open_trades: 8
default_stake_pct: 0.015
max_stake_pct: 0.03
stop_loss_pct: -0.08
```

**Step 4: Restart and check logs**

```bash
docker restart hyperliquid-copy-bot
sleep 5
docker logs hyperliquid-copy-bot --tail 80 2>&1 | grep -iE "error|traceback|failed" || true
```

Expected: no new startup tracebacks/errors.

**Step 5: Verify live settings after restart**

Repeat Step 3.

---

### Task 6: Tighten Invo Reader forwarding filter

**Objective:** Reduce noise by preventing >10x source leverage signals from reaching Aria, while Aria remains the hard gate.

**Files:**
- VPS host file: `/root/apps/invo-reader/.env` — edit carefully, do not print secrets.
- Container: `invo-reader-xvfb`

**Step 1: Change only the non-secret setting**

On the VPS host, update:

```bash
MAX_SOURCE_LEVERAGE_TO_FORWARD=10
```

Do **not** print the full `.env`.

**Step 2: Recreate Invo Reader container using the known-safe absolute-volume pattern**

Do not use `docker:cli compose` with relative paths unless the volume pitfall has been fixed. Recreate the container with the same existing options and explicit absolute host bind mounts, or use the existing deployment method known to preserve:

- `/root/apps/invo-reader/invo_profile:/app/profile`
- `/root/apps/invo-reader/invo_data:/app/data`
- `/root/apps/invo-reader/invo_captures:/app/captures`

**Step 3: Verify env without secrets**

```bash
docker inspect invo-reader-xvfb --format '{{range .Config.Env}}{{println .}}{{end}}' \
  | grep -E '^(ARIA_FILTER_SIDE|MAX_SOURCE_LEVERAGE_TO_FORWARD|SEND_FILTERED_SIGNALS_TO_ARIA|INVO_TELEGRAM_MODE|TELEGRAM_ON_FILTERED_SIGNAL)='
```

Expected:

```text
ARIA_FILTER_SIDE=long
MAX_SOURCE_LEVERAGE_TO_FORWARD=10
SEND_FILTERED_SIGNALS_TO_ARIA=false
INVO_TELEGRAM_MODE=sent_only
TELEGRAM_ON_FILTERED_SIGNAL=false
```

**Step 4: Verify reader state survived**

```bash
docker exec invo-reader-xvfb ls -la /app/data/
curl -s http://127.0.0.1:8090/status | python3 -m json.tool | head -80
```

Expected: existing state files are present; reader is not stuck on welcome/loading screen.

---

### Task 7: Push source repo commits

**Objective:** Ensure the next Docker rebuild preserves the hotfix.

**Files:**
- Repo: `/srv/hermes-repos/hyperliquid-copy-bot/`

**Step 1: Check commits**

```bash
cd /srv/hermes-repos/hyperliquid-copy-bot
git log --oneline --decorate -5
git status --short --branch
```

Expected: defensive commits on `main`, clean working tree.

**Step 2: Push**

```bash
git push origin main
```

**Step 3: Verify branch**

```bash
git status --short --branch
git log --oneline --decorate -3
```

Expected: `HEAD -> main, origin/main` includes the new commits.

---

## Post-Deployment Verification

Run these after deployment.

### Verify no unsafe config survived

```bash
docker exec hyperliquid-copy-bot uv run python - <<'PY'
from app.core.config import get_settings
s = get_settings()
assert s.invo_allowed_traders == 'vortex_legion,bustersquad,blocked'
assert s.invo_max_source_leverage == 10.0
assert s.ai_validation_min_confidence == 0.60
assert s.max_total_exposure_pct == 0.50
assert s.max_open_trades == 8
assert s.default_stake_pct == 0.015
assert s.max_stake_pct == 0.03
assert s.stop_loss_pct == -0.08
print('Defensive config: PASS')
PY
```

### Verify recent copied signals are now restricted

After new signals arrive:

```bash
docker exec hyperliquid-copy-bot uv run python - <<'PY'
import sqlite3, json
conn = sqlite3.connect('/app/data/hyperliquid_copy_bot.db')
conn.row_factory = sqlite3.Row
for row in conn.execute('''
SELECT trader, symbol, source_leverage, decision, reason, created_at
FROM invo_signals
WHERE created_at >= datetime('now', '-6 hours')
ORDER BY created_at DESC
LIMIT 50
'''):
    print(json.dumps(dict(row), default=str))
PY
```

Expected:

- No new copied `kzs`, `luxora`, or `beef131412` signals.
- No copied signals with `source_leverage > 10`.
- Rejected signals show clear reasons.

### Verify AI feedback records real PnL

After the next AI-approved trade closes:

```bash
docker exec hyperliquid-copy-bot uv run python - <<'PY'
import sqlite3, json
conn = sqlite3.connect('/app/data/hyperliquid_copy_bot.db')
conn.row_factory = sqlite3.Row
for row in conn.execute('''
SELECT o.trade_id, o.coin, o.ai_confidence, o.trade_pnl, o.was_profitable,
       t.pnl_amount, t.pnl_pct
FROM ai_validation_outcomes o
JOIN paper_trades t ON t.id = o.trade_id
ORDER BY o.id DESC
LIMIT 10
'''):
    print(json.dumps(dict(row), default=str))
PY
```

Expected: `o.trade_pnl == t.pnl_amount`, not all `0.0`.

---

## Rollback Plan

If Aria becomes too quiet or blocks obviously good signals:

1. Lower AI threshold to `0.55` in `Settings.__init__`.
2. Expand allowlist one trader at a time.
3. Raise source leverage cap from `10` to `15`, not back to `40`.
4. Re-deploy config and restart.

Do **not** rollback the AI feedback bug fix.

---

## Verification Checklist

- [ ] `Settings.__init__` overrides stale env values for defensive defaults.
- [ ] `INVO_ALLOWED_TRADERS` effective value is `vortex_legion,bustersquad,blocked` in the running container.
- [ ] `INVO_MAX_SOURCE_LEVERAGE` effective value is `10.0` in the running container.
- [ ] `AI_VALIDATION_MIN_CONFIDENCE` effective value is `0.60` in the running container.
- [ ] `MAX_TOTAL_EXPOSURE_PCT` effective value is `0.50` in the running container.
- [ ] `MAX_OPEN_TRADES` effective value is `8` in the running container.
- [ ] `DEFAULT_STAKE_PCT` effective value is `0.015` in the running container.
- [ ] `MAX_STAKE_PCT` effective value is `0.03` in the running container.
- [ ] `STOP_LOSS_PCT` effective value is `-0.08` in the running container.
- [ ] Invo Reader effective `MAX_SOURCE_LEVERAGE_TO_FORWARD=10`.
- [ ] AI feedback rows store actual non-zero PnL when a closed trade has non-zero PnL.
- [ ] No startup tracebacks in Aria logs after restart.
- [ ] Source repo commits are pushed to `origin/main`.

---

## Design Decisions

| Decision | Rationale |
|----------|-----------|
| **Hard gates before AI** | The AI approved several losing trades. Deterministic filters should block known-bad buckets before spending API calls. |
| **Allowlist over blocklist** | In early testing, the safer default is to follow only known acceptable traders. A blocklist misses new bad traders. |
| **10x source leverage cap** | Live data shows 6–10x was profitable while 11x+ was strongly negative. 10x is the empirical cut line. |
| **Aria rejects even if Invo forwards** | Invo Reader config can drift. Aria must remain the final enforcement layer. |
| **Runtime overrides in code** | The VPS has stale root-owned env vars. Code overrides avoid fragile root `.env` edits for urgent defensive settings. |
| **No forced close of current trades** | Option A was about stopping bad new entries. Forced liquidation is a separate trading decision Hammad should explicitly approve. |
| **Keep stop-loss modestly tighter** | A stop-loss does not fix bad entries, but `-20%` is too wide for paper evaluation. `-8%` limits tail damage while observing. |

---

## Future Enhancements

- **Trader probation system:** Automatically demote traders after N losing trades or drawdown threshold.
- **Source leverage multiplier:** Instead of hard rejecting all >10x, downsize or require higher AI confidence for 11–20x.
- **Market regime gate:** Pause altcoin longs when BTC/ETH market regime is strongly bearish.
- **Open-position dashboard:** Telegram command/report summarizing open MTM, trader exposure, and risk gates.
- **Backtested exit rules:** Evaluate `-5%`, `-8%`, trailing stops, and time exits on actual Bitget candle data before changing exits again.

---

## Resource Impact

| Resource | Before | After | Delta |
|----------|--------|-------|-------|
| RAM | Baseline Aria + Invo | Same | ~0 MB |
| Storage | SQLite + logs | Same | Tiny increase from tests/docs only |
| CPU | AI research for many signals | Lower | Fewer AI calls because hard gates reject earlier |
| API calls | DeepSeek/Brave per candidate signal | Lower | Fewer copied candidates reach AI |
| Trade count | Up to 20 open trades, broad traders | Up to 8 open trades, 3 traders | Much lower risk and lower signal volume |
