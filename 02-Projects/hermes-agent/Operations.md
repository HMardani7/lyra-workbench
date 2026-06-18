# Hermes Agent — Operations

## Common commands

```bash
hermes doctor
hermes status --all
hermes config
hermes config path
hermes config env-path
hermes tools list
hermes skills list
```

Gateway:

```bash
hermes gateway status
sudo hermes gateway restart
```

From Telegram, `/restart` may also restart the gateway depending on permissions/config.

## Important current env

```text
OBSIDIAN_VAULT_PATH=/home/hermes/workbench
```

## Safety notes

- Config values belong in `config.yaml`; secrets belong in `.env`.
- Do not print `.env` contents.
- Toolset/config changes often require restart or fresh session.
- Telegram/gateway final responses should be concise unless Hammad asks for detail.

## Troubleshooting starting points

- Gateway logs: `/home/hermes/.hermes/logs/gateway.log`
- Config check: `hermes config check`
- Doctor: `hermes doctor`
- Tool availability: `hermes tools list`
