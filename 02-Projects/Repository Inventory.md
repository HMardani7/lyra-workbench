# Repository Inventory

Last reviewed: 2026-06-18

This note lists the git repositories currently readable from the Hermes VPS environment and the matching project note folder in this vault.

## Canonical project repos

### Aria / Hyperliquid Copy Bot

- Repo path: `/srv/hermes-repos/hyperliquid-copy-bot`
- Remote: `HMardani7/hyperliquid-copy-bot`
- Vault folder: `02-Projects/aria/`
- Purpose: Python/FastAPI/CLI trading-signal system for Hyperliquid research, Invo webhook ingestion, Bitget spot-compatible paper execution, risk controls, AI validation, and Telegram alerts.

### Invo Reader

- Repo path: `/srv/hermes-repos/invo-reader`
- Remote: `HMardani7/invo-reader`
- Vault folder: `02-Projects/invo/`
- Purpose: Node/Playwright service that monitors Invo notifications, extracts structured trade details, and forwards filtered long-side signals to Aria.

### Orion

- Repo path: `/srv/hermes-repos/orion-scheduler-assistant`
- Vault folder: `02-Projects/orion/`
- Purpose: local-first productivity app monorepo with Expo React Native mobile app, shared TypeScript package, and Node API for auth/sync/AI/voice/calendar endpoints.

### Closer

- Repo path: `/srv/hermes-repos/closer`
- Remote: `HMardani7/closer`
- Vault folder: `02-Projects/closer/`
- Purpose: long-distance relationship app MVP monorepo scaffold.

### Hermes Agent

- Repo path: `/usr/local/lib/hermes-agent`
- Remote: NousResearch Hermes Agent source install
- Vault folder: `02-Projects/hermes-agent/`
- Purpose: the Hermes agent framework powering Lyra, Telegram gateway, tools, skills, memory, cron, plugins, and subagents.

### Lyra Workbench

- Repo path: `/home/hermes/workbench`
- Remote: `HMardani7/lyra-workbench`
- Vault folder: `02-Projects/lyra-workbench/`
- Purpose: this Obsidian vault and shared agent workbench.

## Non-canonical / duplicate clones

### `/srv/hermes-workbench`

Older/stale clone of `HMardani7/lyra-workbench` on branch `docs/stl-sourcing-report-20260614` with uncommitted work. Do **not** use as canonical vault. Canonical vault is `/home/hermes/workbench`.

### `/srv/hermes-workbench/aria_impl`

Duplicate/stale clone of the Aria/Hyperliquid Copy Bot repository. Use `/srv/hermes-repos/hyperliquid-copy-bot` as canonical read target instead.

## Agent policy

- Treat application repos under `/srv/hermes-repos/` as read-only unless Hammad explicitly asks for code changes.
- Write plans, context, patch drafts, and handoffs in this vault instead.
- Do not read `.env`, logs, databases, auth/session stores, or credential files.
- Start from the relevant project overview/index note, then read targeted repo files only.
