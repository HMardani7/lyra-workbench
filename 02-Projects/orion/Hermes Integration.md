# Orion — Hermes Integration

## Current status

Hermes integration has been planned but backend/agent deployment is deferred until homelab/server deployment. Local USB/ADB testing has worked in prior sessions.

Existing detailed plan:

- `02-Projects/orion/orion-agent-integration.md`

Related Hermes skill:

- `~/.hermes/skills/productivity/orion-agent/SKILL.md`

## Direction

Orion should eventually let Hammad add/manage tasks through Lyra/Hermes, but current repo notes say Obsidian/note integrations are intentionally out of scope inside Orion itself for now.

## Design guardrails

- Keep Orion's mobile local-first model intact.
- Use explicit backend/API boundaries for agent actions.
- Avoid direct mobile database mutation from agents.
- Require human approval for destructive or broad task changes.
- Treat scheduled reminders/calendar sync carefully; production notification delivery remains follow-up work.
