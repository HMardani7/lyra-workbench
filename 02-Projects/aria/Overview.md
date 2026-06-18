# Aria / Hyperliquid Copy Bot — Overview

## Purpose

Aria is Hammad's crypto copy-trading / signal-evaluation system. The current repo name is `hyperliquid-copy-bot`, but the project has evolved beyond simple Hyperliquid wallet copying.

It currently focuses on:
- receiving structured Invo trade signals via webhook
- simulating Bitget spot-compatible paper trades
- applying risk controls, fee/edge checks, portfolio exposure rules, and AI validation
- sending Telegram alerts and reports
- using Hyperliquid wallet data mainly as research/signal context unless Hammad explicitly says otherwise

## Canonical repo

- Path: `/srv/hermes-repos/hyperliquid-copy-bot`
- GitHub: `HMardani7/hyperliquid-copy-bot`
- Main container: `hyperliquid-copy-bot`
- VPS production path from memory: `/root/hyperliquid-copy-bot`

## Stack

- Python 3.11+
- `uv` package manager
- Typer CLI
- FastAPI webhook for Invo signals
- SQLModel / SQLite
- APScheduler daemon jobs
- Telegram bot integration
- Docker Compose deployment

## Current operating mode

- Main mode: `PAPER` / Bitget spot-compatible paper execution.
- Real Bitget live trading is disabled/not implemented.
- Hyperliquid wallet watching is not the trusted copy source anymore; use HL wallet data for research/signal unless Hammad says otherwise.
- Invo Reader is the active signal source for long-side copy research/flow.

## Related notes

- [[Architecture]]
- [[Operations]]
- [[Development Workflow]]
- [[Repository Inventory]]
