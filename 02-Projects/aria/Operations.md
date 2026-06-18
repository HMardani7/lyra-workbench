# Aria — Operations

## Deployment model

Aria runs as a Docker Compose service on the VPS.

Canonical repo for inspection:

```text
/srv/hermes-repos/hyperliquid-copy-bot
```

Production path from Lyra memory:

```text
/root/hyperliquid-copy-bot
```

Container:

```text
hyperliquid-copy-bot
```

Mounted data/log paths in production memory:

```text
/root/hyperliquid-copy-bot/data
/root/hyperliquid-copy-bot/logs
```

## Common commands from repo docs

```bash
docker compose ps
docker compose logs -f
docker compose up -d --build
docker compose restart
```

Inside container examples:

```bash
docker compose exec hyperliquid-copy-bot uv run python -m app.cli recent-signals
docker compose exec hyperliquid-copy-bot uv run python -m app.cli paper-status
docker compose exec hyperliquid-copy-bot uv run python -m app.cli bitget-refresh-symbols
docker compose exec hyperliquid-copy-bot uv run python -m app.cli reset-paper --balance 50
```

## Deployment preference

Hammad prefers GitHub-first gating:

1. inspect/discuss architecture
2. write implementation plan
3. implement in repo branch if requested
4. test thoroughly in container/sandbox
5. push branch / verify
6. only then merge/deploy to VPS with explicit approval

## Safety notes

- Do not print `.env` or secrets.
- Do not enable real trading without explicit approval.
- Do not deploy production changes without explicit approval.
- Treat VPS state as source of truth when it diverges from untested laptop commits.
