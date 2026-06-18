# Hermes Agent — Development Workflow

## Before modifying Hermes itself

Always load/read the Hermes-specific skill/docs first. In Lyra sessions this means using the `hermes-agent` skill before answering setup/configuration questions.

## Source path

```text
/usr/local/lib/hermes-agent
```

## Testing commands from Hermes docs

```bash
python -m pytest tests/ -o 'addopts=' -q
python -m pytest tests/tools/ -q
```

## Contribution rules

- Do not break prompt caching by changing context/tools/system prompt mid-session.
- Preserve message role alternation.
- Use `get_hermes_home()` for profile-safe paths.
- New tools need registry registration, toolset assignment, and readiness checks.
- Config values go in `config.yaml`; secrets go in `.env`.

## Agent caution

This repo powers the currently running agent. Prefer plans/reviews over live code edits unless Hammad explicitly asks for Hermes development work.
