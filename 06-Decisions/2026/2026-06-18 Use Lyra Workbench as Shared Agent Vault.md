# Decision: Use Lyra Workbench as Shared Agent Vault

Date: 2026-06-18
Status: Accepted

## Context

Hammad wanted a longer-term memory layer for Lyra and other agents. We considered using a separate Obsidian vault or a dedicated memory system.

## Decision

Use the existing Lyra Workbench repo as the Obsidian vault and shared agent knowledge base.

## Rationale

- Avoids creating another repo.
- Keeps plans, research, decisions, and agent context in one place.
- Obsidian can open the repo directly.
- Git provides syncing, history, and rollback.
- Markdown files are readable by humans and agents.

## Alternatives Considered

- Separate Obsidian vault repo.
- Dedicated vector database/RAG memory system.
- Hermes built-in memory only.

## Consequences

- The repo needs clear structure to avoid becoming messy.
- Agents should pull before reading and commit/push after writing.
- Durable context should be distilled into notes, not dumped as raw chat logs.
