# Orion — Overview

## Purpose

Orion is Hammad's personal productivity app: a mobile-first, local-first task scheduler/planner with AI/voice assistance, sync foundations, calendar integration foundations, and premium productivity views.

## Canonical repo

- Path: `/srv/hermes-repos/orion-scheduler-assistant`
- Workbench plans: `03-Implementation-Plans/Orion/` and existing `02-Projects/orion/orion-agent-integration.md`

## Stack

- Expo React Native + Expo Router mobile app
- TypeScript monorepo with npm workspaces
- `packages/shared` for contracts, parsers, date/scheduling helpers, sync merge helpers
- `services/api` lightweight Node API for auth, sync, AI/voice, external calendar endpoints
- Local-first native persistence via `expo-sqlite`
- Web preview fallback via in-memory/localStorage repo

## Current product areas

- Inbox, Today, Urgent, Calendar, Projects, Search, Saved Filters, Completed Archive
- Task detail editing with priority/due/deadline/duration/recurrence/labels/reminders/subtasks
- Voice dump capture/review/accept flow
- AI assist flows: breakdown, actionable rewrite, completion tips, daily plan, overdue cleanup
- Android urgent widget
- Auth foundation and protected API endpoints
- Cloud sync foundation with LWW, tombstones, conflict logs
- External calendar Google-first read-only foundation
- Premium settings/control center and onboarding

## Related notes

- [[Architecture]]
- [[Development Workflow]]
- [[Hermes Integration]]
