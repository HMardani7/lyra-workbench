# Closer — Architecture

## Repository layout

```text
apps/frontend   # frontend app; intentionally unscaffolded for now
apps/backend    # backend API scaffold
infra           # deployment/infrastructure assets
scripts         # utility scripts
docs            # product, architecture, API contracts, workflow
```

## Architecture direction

- Monorepo to evolve frontend/backend/infra/docs together while preserving boundaries.
- Backend exposes API endpoints consumed by frontend.
- Frontend/backend stay decoupled through explicit API contracts in `docs/API_CONTRACTS.md`.
- Future deployment likely targets AWS, but cloud details are intentionally not hardcoded yet.
- Data layer may use DynamoDB or another managed database, but backend should remain storage-agnostic during MVP.

## Backend scaffold

- Python backend scaffold in `apps/backend`.
- Minimal FastAPI-style app entrypoint.
- No business features implemented yet.

## Architectural principle

Keep architecture flexible during MVP; prioritize reliable core journey over premature advanced features.
