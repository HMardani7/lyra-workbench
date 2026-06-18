# Hermes Agent — Overview

## Purpose

Hermes Agent is the open-source AI agent framework powering Lyra on the VPS/Telegram. It provides chat, tool use, skills, memory, session search, gateway messaging integrations, cron jobs, plugins, subagents, model/provider routing, and terminal/file/web tooling.

## Canonical repo/source path

- Path: `/usr/local/lib/hermes-agent`
- User-facing docs: https://hermes-agent.nousresearch.com/docs
- Active profile: default under `/home/hermes/.hermes`

## Major subsystems

- `agent/` — model routing, prompt building, memory, compression, tool execution, transports, providers.
- `tools/` — tool implementations and registry.
- `gateway/` — Telegram/Discord/etc messaging gateway.
- `hermes_cli/` — CLI commands/config/setup.
- `cron/` — scheduled jobs.
- `plugins/` — optional integrations.
- `optional-skills/` and `skills/` conventions — reusable procedural knowledge.

## Current Lyra setup

- Telegram gateway is active.
- `OBSIDIAN_VAULT_PATH=/home/hermes/workbench` points Hermes to Lyra Workbench.
- User-facing persona is Lyra.
- Hammad uses this primarily from Telegram/mobile.

## Related notes

- [[Architecture]]
- [[Operations]]
- [[Development Workflow]]
