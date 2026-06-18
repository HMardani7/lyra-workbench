# Lyra Workbench

This repo is Hammad's shared agent workbench and Obsidian vault.

It contains:
- durable project knowledge
- implementation plans
- research reports
- agent handoffs
- decision records
- generated artifacts

## Key folders

- `00-Inbox/` — unsorted capture and quick notes
- `01-Knowledge/` — stable shared context for all agents
- `02-Projects/` — project-specific source of truth
- `03-Implementation-Plans/` — execution-ready specs
- `04-Research/` — investigations and reports
- `05-Agent-Handoffs/` — tasks prepared for Codex/Kiro/Lyra
- `06-Decisions/` — decision records and rationale
- `07-Templates/` — reusable note templates
- `08-Archive/` — old/superseded material
- `09-Artifacts/` — generated outputs

## Agent rule

Before working on a project, agents should read:
1. `01-Knowledge/Context/Preferences.md`
2. `01-Knowledge/Context/Environment.md`
3. `01-Knowledge/Context/Active Work.md`
4. `01-Knowledge/Context/Vault Operating Rules.md`
5. relevant overview/index files in `02-Projects/<Project>/`

Agents should **not** bulk-read the vault. Search first, then read targeted files.

## Token/context discipline

This repo is meant to prevent repeated context, not create a huge context burden.

- Keep `01-Knowledge/` concise and durable.
- Put long exploratory work in `04-Research/` with a short README/index.
- Move stale/superseded content to `08-Archive/`.
- Do not store raw logs or chat dumps unless there is a strong reason.
- Distill stable conclusions into project/context notes.

See `01-Knowledge/Context/Vault Operating Rules.md` for the full policy.
