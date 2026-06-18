# Aria — Architecture

## High-level flow

```text
Invo Reader
  → POST /signals/invo
  → Aria webhook auth/validation
  → Invo signal processor
  → risk checks + portfolio context + AI validation
  → paper execution / ledger updates
  → Telegram alerts + reports
```

Historical Hyperliquid wallet ingestion still exists, but Hammad no longer treats Hyperliquid wallets as reliable copy sources.

## Important modules

- `app/invo/webhook.py` — FastAPI endpoint `POST /signals/invo`; validates webhook secret and schema.
- `app/invo/processor.py` — converts Invo payloads into Aria trading decisions.
- `app/invo/models.py`, `app/invo/schemas.py` — Invo domain models/contracts.
- `app/paper/*` — spot-compatible paper execution, PnL, matching, portfolio state.
- `app/risk/*` — safety/risk policy: leverage, exposure, fee edge, coin risk, market regime, stale positions, stake sizing.
- `app/ai/*` — AI validator/research/feedback loop.
- `app/analytics/*` — decision quality and trader advisory logic.
- `app/telegram/*` — command and alert interfaces.
- `app/hyperliquid/*` — wallet/fill/position ingestion and signal detection.
- `app/jobs/*` — scheduler/daemon tasks.
- `app/db/*` — SQLite database/session/models.

## Webhook contract

Endpoint:

```text
POST /signals/invo
Header: X-Aria-Webhook-Secret
```

Accepted actions:

```text
open, close, update, increase, decrease
```

Accepted sides:

```text
long, short
```

The payload must include trader, symbol, and stable `position_key` so close/update events can match the original open.

## Safety defaults worth preserving

- `global_kill_switch` exists.
- `allow_add_to_position` defaults false.
- `paper_spot_require_live_compatible` exists to keep paper decisions compatible with eventual spot execution.
- Bitget live trading is not implemented and should not be enabled casually.
- AI timeout fallback is configured conservatively; avoid accidental approve-on-timeout behaviour.

## Data and persistence

- SQLite database mounted under Docker data volume.
- Logs mounted separately.
- Daily backups exist in deployment docs.

## Known duplicate clone

`/srv/hermes-workbench/aria_impl` appears to be a duplicate/stale clone of this repo. Prefer canonical path `/srv/hermes-repos/hyperliquid-copy-bot`.
