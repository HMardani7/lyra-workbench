# Hermes Agent — Architecture

## High-level loop

```text
message source (CLI/Telegram/etc)
  → session/context builder
  → model call with tool schemas
  → tool dispatch loop
  → final response/delivery
```

## Key areas

- `run_agent.py` / conversation loop code — core agent turn loop.
- `agent/prompt_builder.py` — system prompt/environment hints/context assembly.
- `agent/tool_executor.py` and related helpers — tool call dispatch.
- `tools/registry.py` + `tools/*.py` — tool registration and handlers.
- `gateway/` — messaging platforms and delivery plumbing.
- `hermes_cli/commands.py` — slash command registry.
- `hermes_cli/config.py` — default config/env mappings.
- `cron/` — durable scheduled jobs.
- `agent/memory_*` — memory provider abstractions.
- `agent/skill_*` — skill loading/preprocessing/commands.

## Profile/config paths

- Main config: `/home/hermes/.hermes/config.yaml`
- Env/secrets: `/home/hermes/.hermes/.env`
- Skills: `/home/hermes/.hermes/skills/`
- Sessions DB: `/home/hermes/.hermes/state.db`
- Logs: `/home/hermes/.hermes/logs/`

## Obsidian integration

Hermes accesses the vault as a filesystem repo via:

```text
OBSIDIAN_VAULT_PATH=/home/hermes/workbench
```

Agents should use `git pull` before reading and commit/push after writing durable vault changes.
