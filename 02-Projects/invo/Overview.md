# Invo Reader — Overview

## Purpose

Invo Reader is a standalone Node/Playwright service that monitors Invo trading-platform notifications, extracts structured trade details, and forwards filtered signals to Aria by webhook.

It is observational only:

- it does not place trades
- it does not click Mimic Trade
- it does not execute orders

## Canonical repo

- Path: `/srv/hermes-repos/invo-reader`
- GitHub: `HMardani7/invo-reader`
- Production container from memory: `invo-reader-xvfb`

## Stack

- Node.js >= 18
- Playwright Chromium
- Express health/status server
- Docker Compose
- Xvfb/noVNC support for VPS/manual-login flows

## Why it exists

Invo is a Flutter Canvas web app, so normal DOM scraping does not work. The service uses browser automation and network/API interception to capture notifications and post details.

## Related notes

- [[Architecture]]
- [[Operations]]
- [[Development Workflow]]
- [[Aria]]
