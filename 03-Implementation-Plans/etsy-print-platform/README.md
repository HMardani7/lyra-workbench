# Etsy Print Platform — Kiro Handoff

**Status:** Architecture draft for review before implementation.

**Goal:** Build a small local-first platform that turns Etsy orders for 3D-printed products into safe, trackable Bambu P1S print jobs.

**Read order for Kiro:**
1. `00-system-architecture.md` — domain model, Etsy API flow, printer flow, safety gates.
2. `01-etsy-api-notes.md` — exact Etsy endpoints, OAuth scopes, response fields, webhook shape.
3. `02-printer-integration-notes.md` — Bambu P1S LAN/FTPS/MQTT integration assumptions.
4. `03-kiro-build-prompt.md` — implementation prompt and staged tasks.

## Assumed hardware

- Printer: **Bambu Lab P1S**
- Material handling: **AMS-powered multi-material printing**
- Initial nozzle: **0.4mm**
- Product type: physical Etsy goods printed on demand from pre-vetted STL/3MF assets.

## Core principle

The system should **not auto-print unknown customer input**. It should automate boring workflow steps, but production remains safe:

- listings are created from approved product templates
- orders are imported from Etsy
- print jobs are queued against known SKUs/variants
- printer state and filament requirements are checked
- operator confirms before starting a print, at least for v1
- shipment tracking is posted back to Etsy after labels are purchased outside the API

## Important Etsy limitation

Etsy Open API v3 can update receipts with shipment tracking, but the fulfillment tutorial says the API **does not include an endpoint to purchase shipping labels**. The platform should therefore record label/tracking details entered manually or imported from a shipping tool, then call Etsy's tracking endpoint.

## Open questions for Hammad

- Exact shipping workflow: Royal Mail Click & Drop, Etsy labels manually, Evri, DPD, or something else?
- Whether printer control should be full auto-start or “prepare job + ask Hammad to confirm.” Recommendation: confirm for v1.
- Whether the MVP app should be local-only on the homelab or cloud-accessible via tunnel.
