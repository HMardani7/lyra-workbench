# Invo Trader Discovery + Hyperliquid Removal — Implementation Plan

> **For Hermes:** Use subagent-driven-development skill to implement this plan task-by-task.

**Goal:** Remove the dead Hyperliquid wallet watching pipeline (zero active wallets, log spam every 15 seconds) and add an Invo trader discovery feature that surfaces new traders worth following.

**Architecture:** Two independent changes in one plan: (1) Strip Hyperliquid — remove the watcher, signal detector, fill aggregation, and related DB tables from the tick loop. (2) Add a `discover_invo_traders()` function that polls the Invo reader `/status` endpoint, compares against currently-tracked traders, and surfaces candidates in the daily report.

**Tech Stack:** Python 3.11+, httpx, SQLModel, Pydantic Settings, pytest, SQLite

---

## PART A: Hyperliquid Removal

### What This Changes

Hyperliquid wallet watching is dead weight: "No active wallets to fetch" fires ~5,760 times/day. All signals come from Invo. Removing it:

- **Eliminates log spam** — 5,760 fewer log lines per day
- **Frees CPU** — no more pointless API calls to Hyperliquid info endpoint
- **Cleans the codebase** — ~5 modules simplified or removed
- **Cleans the DB** — 6 empty tables no longer queried

### What Gets Removed/Stopped

| Component | Action |
|-----------|--------|
| `app/jobs/tasks.py` — `fetch_wallet_data_task()` | **Stop calling** (remove from scheduler) |
| `app/jobs/tasks.py` — `group_and_detect_task()` | **Stop calling** (remove from scheduler) |
| `app/hyperliquid/watcher.py` | Keep file (for reference), remove import from daemon |
| `app/hyperliquid/aggregation.py` | Keep file, remove import |
| `app/hyperliquid/signal_detector.py` | Keep file, remove import |
| DB tables: `watched_wallets`, `wallet_snapshots`, `hyperliquid_fills`, `fill_groups`, `detected_signals`, `hyperliquid_position_snapshots` | **Don't drop** (keep for historical data), stop querying |

**What stays:** `app/hyperliquid/` module files remain on disk (they might be useful to reference). They're just not called from the daemon anymore.

**What about `process_all_signals()`?** It queries `detected_signals` where `status == "new"`. With no wallet watcher, this table gets no new rows — the query returns empty. No change needed. The function still handles Invo signals (they bypass this code path).

### Files Modified

| File | Action | Purpose |
|------|--------|---------|
| `app/jobs/scheduler.py` | Modify | Remove `fetch_wallet_data_task` and `group_and_detect_task` from the schedule |
| `app/jobs/daemon.py` | Modify | Remove any direct wallet watcher import/calls |

---

## PART B: Invo Trader Discovery

### What This Upgrade Does

You manually pick Invo traders to follow. There's no discovery — you don't know who else is available or how they're performing. The Invo reader has a `/status` endpoint that lists all active traders with their stats (signal count, recent PnL, etc.). This feature polls that endpoint weekly and surfaces promising traders you're NOT already copying.

### How It Works

1. Poll `INVO_READER_STATUS_URL` (already configured: `http://172.17.0.1:8090/status`)
2. Parse the response for all available traders and their public stats
3. Compare against traders in `invo_allowed_traders`
4. Rank unknown traders by recent performance (win rate, signal count, PnL)
5. Add a "Trader Discovery" section to the weekly report

### Files to Touch

| File | Action | Purpose |
|------|--------|---------|
| `app/invo/discovery.py` | **Create** | Trader discovery logic |
| `app/jobs/tasks.py` | Modify | Add `trader_discovery_task()` (runs weekly) |
| `app/jobs/scheduler.py` | Modify | Schedule the weekly task |
| `app/reports/daily.py` | Modify | Add discovery section to daily report |
| `tests/test_invo_discovery.py` | **Create** | Tests for discovery logic |

---

## Task 1 (Hyperliquid): Stop the scheduler tasks

**File:** `app/jobs/scheduler.py`

Find the schedule configuration. Remove or comment out these two tasks:

```python
# BEFORE:
schedule.every(settings.poll_interval_seconds).seconds.do(fetch_wallet_data_task)
schedule.every(settings.poll_interval_seconds).seconds.do(group_and_detect_task)

# AFTER — keep these lines but add them to a DISABLED section or remove:
# Hyperliquid wallet watching disabled — all signals now come from Invo.
# schedule.every(settings.poll_interval_seconds).seconds.do(fetch_wallet_data_task)
# schedule.every(settings.poll_interval_seconds).seconds.do(group_and_detect_task)
```

**Verify:**

```bash
# Restart Aria, check logs — no more "No active wallets to fetch" spam
docker logs hyperliquid-copy-bot 2>&1 | grep -c "No active wallets"
# Expected: 0
```

---

## Task 2 (Hyperliquid): Clean daemon imports

**File:** `app/jobs/daemon.py`

If the daemon directly imports the wallet watcher, remove the import. Usually the daemon just calls the scheduler — verify no direct `WalletWatcher` references remain.

---

## Task 3 (Discovery): Create poller

**File:** `app/invo/discovery.py`

```python
"""Invo trader discovery — finds new traders worth following."""

from __future__ import annotations

import httpx
from loguru import logger

from app.core.config import get_settings


def discover_traders() -> list[dict]:
    """Poll the Invo reader status endpoint and return undiscovered traders.

    Returns a list of trader dicts sorted by recent performance (best first).
    Each dict: {trader, signal_count, win_rate_7d, avg_pnl, status}
    """
    settings = get_settings()
    url = settings.invo_reader_status_url

    if not url:
        logger.warning("Invo reader status URL not configured")
        return []

    try:
        response = httpx.get(url, timeout=10.0)
        response.raise_for_status()
        data = response.json()
    except Exception as e:
        logger.error(f"Failed to fetch Invo trader status: {e}")
        return []

    # Parse traders — the exact response shape depends on the Invo reader API.
    # Typical: {"traders": [{"name": "...", "signals": N, "win_rate_7d": 0.X, ...}]}
    all_traders = data.get("traders", [])
    if not all_traders:
        # Alternative: maybe the key is different
        all_traders = data.get("active_traders", data.get("sources", []))

    if not all_traders:
        logger.debug("No traders in Invo reader status response")
        return []

    # Get currently allowed traders
    allowed_raw = settings.invo_allowed_traders.strip()
    if allowed_raw in ("*", "ALL", "all", ""):
        allowed_set: set[str] = set()
    else:
        allowed_set = {t.strip().lower() for t in allowed_raw.split(",")}

    # Find undiscovered traders
    undiscovered = []
    for t in all_traders:
        name = t.get("name", t.get("trader", ""))
        if not name:
            continue
        if name.lower() in allowed_set:
            continue  # Already following

        undiscovered.append({
            "trader": name,
            "signal_count": t.get("signal_count", t.get("signals", 0)),
            "win_rate_7d": t.get("win_rate_7d", t.get("win_rate", 0)),
            "avg_pnl": t.get("avg_pnl", t.get("avg_return", 0)),
            "status": t.get("status", "unknown"),
        })

    # Rank by win rate × signal count (active + good)
    undiscovered.sort(
        key=lambda x: (x["win_rate_7d"] or 0) * min(x["signal_count"], 50),
        reverse=True,
    )

    return undiscovered


def generate_discovery_report() -> str:
    """Generate a trader discovery section for the daily report."""
    traders = discover_traders()

    if not traders:
        return ""

    lines = [
        "\n## 🔍 Trader Discovery\n",
        f"Found {len(traders)} traders you're not currently copying:\n",
        f"{'Trader':<16} {'Signals':>8} {'Win Rate':>9} {'Avg PnL':>10}",
        "-" * 48,
    ]

    for t in traders[:10]:  # Top 10
        wr = t["win_rate_7d"]
        wr_str = f"{wr*100:.0f}%" if isinstance(wr, (int, float)) else str(wr)
        lines.append(
            f"{t['trader']:<16} {str(t['signal_count']):>8} "
            f"{wr_str:>9} {str(t['avg_pnl']):>10}"
        )

    lines.append("\n_To add a trader, update INVO_ALLOWED_TRADERS in .env_")
    return "\n".join(lines)
```

---

## Task 4 (Discovery): Add weekly task

**File:** `app/jobs/tasks.py`

Add at the bottom:

```python
def trader_discovery_task() -> None:
    """Task: Discover new Invo traders and send report."""
    try:
        from app.invo.discovery import discover_traders
        from app.telegram.messages import send_message_sync

        traders = discover_traders()
        if not traders:
            logger.debug("No new traders discovered.")
            return

        top5 = traders[:5]
        msg_lines = ["🔍 *Weekly Trader Discovery*", ""]
        for t in top5:
            wr = t["win_rate_7d"]
            wr_str = f"{wr*100:.0f}%" if isinstance(wr, (int, float)) else str(wr)
            msg_lines.append(
                f"• {t['trader']} — {t['signal_count']} signals, "
                f"{wr_str} win rate, avg PnL {t['avg_pnl']}"
            )
        msg_lines.append("")
        msg_lines.append("To add: update INVO_ALLOWED_TRADERS in .env")

        send_message_sync("\n".join(msg_lines))
        logger.info(f"Trader discovery: found {len(traders)}, reported top 5.")
    except Exception as e:
        logger.error(f"trader_discovery_task failed: {e}")
```

**File:** `app/jobs/scheduler.py`

Add to schedule:

```python
# Weekly trader discovery — runs every Monday at 09:00 UTC
schedule.every().monday.at("09:00").do(trader_discovery_task)
```

---

## Task 5: Write tests

**File:** `tests/test_invo_discovery.py`

```python
class TestTraderDiscovery:
    def test_empty_response_returns_empty(self):
        """Empty/malformed response → empty list."""
        ...

    def test_all_traders_already_followed(self):
        """All traders in response are in allowed list → empty."""
        ...

    def test_new_traders_ranked_by_performance(self):
        """Undiscovered traders sorted by win_rate × signal_count."""
        ...

    def test_api_error_graceful(self):
        """HTTP error → empty list, no crash."""
        ...

    def test_wildcard_skips_all(self):
        """allowed_traders='*' → all traders considered 'already following'."""
        ...
```

---

## Verification Checklist

### Hyperliquid Removal
- [ ] Scheduler no longer runs `fetch_wallet_data_task`
- [ ] Scheduler no longer runs `group_and_detect_task`
- [ ] Container logs have 0 "No active wallets to fetch" messages
- [ ] `process_all_signals()` still works (returns empty counts, no crash)
- [ ] Invo signals still processed normally
- [ ] No import errors on restart

### Trader Discovery
- [ ] Invo reader URL is configured (check `.env`)
- [ ] `/status` endpoint returns valid JSON
- [ ] Discovery correctly filters out existing allowed traders
- [ ] Discovery ranks unknown traders by performance
- [ ] Weekly task runs and sends Telegram message
- [ ] No crash if Invo reader is unreachable
