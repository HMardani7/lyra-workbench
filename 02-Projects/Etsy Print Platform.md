# Etsy Print Platform

## Purpose

Local-first platform to run Hammad's Etsy 3D-print-on-demand workflow around a Bambu Lab P1S with AMS.

## Current source of truth

Implementation/handoff docs live at:

- `03-Implementation-Plans/etsy-print-platform/README.md`
- `03-Implementation-Plans/etsy-print-platform/00-system-architecture.md`
- `03-Implementation-Plans/etsy-print-platform/01-etsy-api-notes.md`
- `03-Implementation-Plans/etsy-print-platform/02-printer-integration-notes.md`
- `03-Implementation-Plans/etsy-print-platform/03-kiro-build-prompt.md`

## Architecture summary

- Product catalog maps approved STL/3MF assets to Etsy SKUs and variants.
- Etsy API creates draft listings, uploads images, manages inventory, imports orders, and posts tracking.
- Etsy webhooks should ingest `order.paid`; a polling fallback should read paid/unshipped receipts.
- Printer integration targets local Bambu P1S FTPS upload + MQTT status/control.
- v1 must require manual operator approval before starting any print.
- Shipping labels are outside Etsy Open API; the app records tracking and posts it back to Etsy.

## Open questions

- Shipping provider/workflow for labels and tracking.
- Exact repo/path for Kiro implementation.
- Whether MVP should be local-only or exposed via a secure tunnel.
