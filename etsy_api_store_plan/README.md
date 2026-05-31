# Etsy Print Farm Automation

> **Build a lights-out Etsy-to-printer pipeline.** Orders come in from Etsy → system matches them to pre-sliced GCODE files → sends to Bambu Lab P1S printer → tracks orders and profit. Zero human touch between "order received" and "print finished."

## What you're building

A Python automation daemon that:

1. **Polls Etsy** for new orders every N minutes via the Etsy API v3
2. **Reads order details** — listing ID + variations (e.g., "Size: 22 inch") selected by the customer
3. **Maps to GCODE files** — uses a YAML config to resolve `listing_id + property_values` → exact file path on disk
4. **Sends to printer** — FTPS upload to Bambu Lab P1S, starts the print via LAN HTTP API
5. **Tracks everything** — SQLite database for orders, revenue, fees, material costs, profit

## Files in this plan

| File | What it is |
|------|-----------|
| `00-overview.md` | Architecture, data flow, folder structure, database schema — read this FIRST |
| `01-implementation-plan.md` | 11 bite-sized TDD tasks with complete code, tests, and commands |
| `02-setup-guide.md` | How to get Etsy API credentials, Bambu Lab access code, and wire it all up |

## How to implement this

### For Kiro / Codex / AI agents

1. **Read `00-overview.md` first** — understand the architecture and "why"
2. **Follow `01-implementation-plan.md`** task-by-task, in order
3. Each task follows TDD: write test → run (fail) → implement → run (pass) → commit
4. All code is copy-paste ready. All commands are exact. All file paths are specified.
5. After all 11 tasks, run `pytest tests/ -v` — all tests should pass
6. See `02-setup-guide.md` for wiring up real Etsy and printer credentials

### For humans

```bash
# 1. Clone or create the project
mkdir etsy-store && cd etsy-store

# 2. Follow 01-implementation-plan.md task by task
#    (or have an AI agent do it)

# 3. Set up credentials (see 02-setup-guide.md)
cp .env.example .env
# Edit .env with your real values

# 4. Create your products.yaml mapping
# Edit products.yaml with your Etsy listing IDs and GCODE file paths

# 5. Run it
python -m etsy_store poll --once    # Test poll
python -m etsy_store metrics        # View dashboard
python -m etsy_store poll           # Run continuously
```

## Tech stack

- **Python 3.11+** — language
- **SQLite + SQLModel** — database (zero config, one file)
- **requests** — Etsy API v3 REST client
- **ftplib** (stdlib) — FTPS file upload to printer
- **PyYAML** — product mapping config
- **paho-mqtt** — printer status monitoring (future)
- **Typer** — CLI
- **pytest** — testing
- **ruff** — linting

## Prerequisites

- Python 3.11 or later (`python --version`)
- An Etsy shop with API credentials (see `02-setup-guide.md`)
- A Bambu Lab P1S printer on your LAN
- Pre-sliced GCODE files organized in product folders
