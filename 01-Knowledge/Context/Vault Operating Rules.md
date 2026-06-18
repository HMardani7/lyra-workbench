# Vault Operating Rules

Purpose: keep Lyra Workbench useful as a shared agent memory system without becoming a giant token sink.

## Core principle

This vault is **not** a chat transcript archive and not a dumping ground. It is a curated knowledge base.

Agents should store:
- durable project facts
- architecture and deployment context
- decisions and rationale
- implementation plans
- distilled research conclusions
- handoff briefs for other agents

Agents should avoid storing:
- raw chat logs
- massive command outputs
- unfiltered web dumps
- temporary task progress that will be stale within a week
- secrets, API keys, tokens, passwords, or copied `.env` contents

## Context budget rules

Before doing project work, agents should read only the minimum useful context:

1. `01-Knowledge/Context/Preferences.md`
2. `01-Knowledge/Context/Environment.md`
3. `01-Knowledge/Context/Active Work.md`
4. this file
5. the relevant project overview/index under `02-Projects/<Project>/`
6. only the specific plan/research/decision files needed for the task

Do **not** bulk-read the whole vault.
Do **not** recursively ingest every markdown file.
Use file search first, then read targeted files.

## Folder reading policy

- `01-Knowledge/` — high-signal, durable context. Keep concise.
- `02-Projects/` — project source of truth. Prefer overview/index files plus targeted detail docs.
- `03-Implementation-Plans/` — execution specs. Read only the relevant plan.
- `04-Research/` — may contain long reports. Read README/index summaries first; open full reports only when needed.
- `05-Agent-Handoffs/` — task briefs. Read only the handoff being executed.
- `06-Decisions/` — decision records. Search by topic/date; read relevant records only.
- `08-Archive/` — old/superseded. Do not read unless explicitly needed.
- `09-Artifacts/` — generated outputs. Do not read unless explicitly needed.

## Note size rules

Aim for:
- Source-of-truth notes: under ~150 lines when possible.
- Overview/index notes: under ~100 lines.
- Research reports: can be longer, but must have a short summary/index at the top or adjacent README.
- Huge outputs/logs: store as artifacts only if necessary, and summarize them in a small markdown note.

When a note gets too large, split it into:
- an overview/index note
- focused detail notes
- archived raw material if needed

## Required summary block for long notes

Any note over ~200 lines should start with:

```md
## Executive Summary

- Key takeaway 1
- Key takeaway 2
- Key takeaway 3

## When to read this full note

Read this only if...
```

## Research lifecycle

1. Long exploratory research goes in `04-Research/`.
2. Stable conclusions get distilled into `01-Knowledge/` or `02-Projects/`.
3. Old raw research moves to `08-Archive/` when superseded.

## Agent write workflow

Before writing:
1. `git pull --ff-only` if safe.
2. Read the relevant existing note first.
3. Make small, focused edits.

After writing:
1. Check `git diff --stat`.
2. Commit with a clear message.
3. Push.

## Linking

Use Obsidian wikilinks for important relationships, for example:
- [[Preferences]]
- [[Environment]]
- [[Active Work]]
- [[Use Lyra Workbench as Shared Agent Vault]]

Prefer links to canonical notes over duplicating the same context in many places.
