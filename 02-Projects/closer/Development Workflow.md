# Closer — Development Workflow

## Folder ownership

- Frontend work: `apps/frontend`
- Backend work: `apps/backend`
- Infrastructure: `infra`
- Shared planning/contracts: `docs`

## Coordination rules

- Use feature branches for non-trivial work.
- Keep PRs focused to one concern where possible.
- Keep frontend/backend integration aligned via `docs/API_CONTRACTS.md`.
- Update API contracts whenever request/response expectations change.
- Avoid mixing frontend and backend changes in the same PR unless necessary.

## Backend run command from repo docs

```bash
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

## Next implementation areas

- authentication and onboarding APIs
- partner linking flows
- couple home/signal/availability/daily question endpoints
- persistence and repository implementations

## Agent guidance

Assign coding agents narrow folder scope such as:

- `apps/backend/app/api`
- `apps/frontend`
- `infra/terraform`
- `docs`
