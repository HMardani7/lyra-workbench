# Invo Reader — Architecture

## High-level flow

```text
Playwright Chromium session
  → capture Invo API auth token / refreshed token
  → direct API polling for notifications
  → detail extraction from post API responses
  → dedupe + persisted seen state
  → filter side/leverage/actions
  → POST structured payload to Aria webhook
```

## Critical decisions

- Direct API calls via `page.evaluate(fetch(...))` are preferred over relying on Flutter UI/network reloads.
- Auth token management is critical: capture Bearer token, capture refresh responses, proactively refresh before expiry.
- Full session keepalive every ~3 hours prevents server-side session expiry.
- Fatal browser errors require browser restart, not just retry.
- Dedupe uses notification hash and timestamp watermark.
- Pagination continues until reaching last processed timestamp.

## Important files

- `src/watcher.js` — main polling loop, API calls, filtering, Aria forwarding.
- `src/browser.js` — browser lifecycle and session save.
- `src/browser-launch.js` — shared launch helper for headless/Xvfb.
- `src/detail-extractor.js` — post-detail navigation and structured trade extraction.
- `src/aria-webhook.js` — signal payload builder, filters, POST to Aria.
- `src/store.js` — seen state, JSONL writing, last-processed tracking.
- `src/config.js` — env parsing and runtime configuration.
- `src/server.js` — Express `/health` and `/status`.
- `docker/start-with-xvfb.sh` — starts Xvfb before commands in VPS mode.

## Filtering before Aria

Configured filters include:

- side filter, e.g. long-only
- max source leverage
- blocked action types
- seen/timestamp dedupe

The stable `source_tracking.chosen_position_key` / `position_key` is essential for Aria to match close/update events to original opens.

## Persisted files

- `invo_data/seen_notifications.json` — dedupe; deleting can replay old notifications.
- `invo_data/reader_state.json` — timestamp/service state; deleting can replay old notifications.
- `invo_profile/storage_state.json` — browser session; deleting requires re-login.
- JSONL/raw notification detail files are historical/debug data.
