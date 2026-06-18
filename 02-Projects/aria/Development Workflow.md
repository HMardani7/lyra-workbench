# Aria — Development Workflow

## Before work

Read:

1. `02-Projects/aria/Overview.md`
2. `02-Projects/aria/Architecture.md`
3. `02-Projects/aria/Operations.md`
4. relevant implementation plans under `02-Projects/aria/` or `03-Implementation-Plans/Aria/`

Then inspect only the target code modules.

## Local/sandbox commands

From repo root:

```bash
uv sync
uv run python -m app.cli --help
uv run pytest
uv run ruff check .
uv run mypy app
```

Docker-oriented checks:

```bash
docker compose build
docker compose up -d
docker compose logs -f
```

## Testing strategy

Use focused tests first:

```bash
uv run pytest tests/unit/test_invo_discovery.py
uv run pytest tests/unit/test_ai_validator.py
uv run pytest tests/unit/test_paper_engine.py
uv run pytest tests/unit/test_stake_sizing.py
```

Then broader suite before deploy.

## Agent rules

- Plans belong in this vault unless Hammad explicitly asks for code edits.
- Keep secrets out of notes and tool output.
- For any production-impacting change, include rollback and verification steps.
