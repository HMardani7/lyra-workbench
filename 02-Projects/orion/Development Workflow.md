# Orion — Development Workflow

## Context read order

The repo already has a context-compression protocol. Use it:

1. Read repo `AGENTS.md` first.
2. If needed, read `SESSION_NOTES.md`, `README.md`, and `docs/architecture.md`.
3. Read only task-specific feature folders/files.
4. If design implications exist, read relevant ADRs in `docs/adr`.

## Useful commands

From repo root:

```bash
npm run mobile
npm run api
npm run android:sim
npm run android:devclient
npm run android:repair
npm run typecheck
npm run lint
npm run test
```

Workspace-specific examples:

```bash
npm --workspace apps/mobile run typecheck
npm --workspace services/api run test
```

## Android runtime notes

Preferred dev-client flow is:

```bash
npm run android:devclient
```

Recovery flow:

```bash
npm run android:repair
```

## Agent rules

- Keep mobile/backend/shared boundaries clean.
- Do not put secrets in mobile env; mobile only gets public env vars.
- Plans for Hermes integration should stay in Lyra Workbench until implementation is explicitly requested.
