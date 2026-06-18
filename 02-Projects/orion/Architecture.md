# Orion — Architecture

## Monorepo layout

- `apps/mobile` — Expo/React Native app, UI, local-first data layer, Android widget integration.
- `services/api` — backend auth/sync/AI/voice/calendar endpoints.
- `packages/shared` — shared entity types, request/response validation, scheduling/date helpers, sync merge helpers.
- `scripts` — Android emulator/dev-client helpers.
- `docs/adr` — architecture decision records.

## Data architecture

- Mobile storage boundary is repository interface under `apps/mobile/src/data/repositories`.
- Native persistence uses SQLite via `expo-sqlite`.
- Migrations use `PRAGMA user_version`.
- Web preview uses `orionRepository.web.ts` with in-memory/localStorage fallback.
- UI state is accessed via `apps/mobile/src/features/tasks/useOrionData.tsx`.

## Backend architecture

`services/api/src/server.ts` defines endpoints for:

- health
- auth signup/login/refresh/logout/me
- voice dump
- AI assistance
- sync push/pull
- calendar connection and event APIs

AI/voice endpoints validate via `@orion/shared`; they use OpenAI when configured and deterministic mock responses when no key exists or calls fail.

## Sync/auth

- Backend sync uses persistent SQLite store.
- Sync requires bearer auth and user scoping.
- LWW conflict handling and tombstones are implemented.
- Auth hashes passwords using `scrypt` and stores hashed opaque tokens.

## Important constraints

- Mobile can only use public `EXPO_PUBLIC_*` variables.
- Secrets stay backend-only.
- External calendar writeback is intentionally disabled.
- Obsidian/note integrations are intentionally out of scope in the repo itself for now.
