# Kiro Build Prompt — Etsy Print Platform MVP

You are Kiro, implementing an MVP for Hammad's Etsy 3D-print platform.

## Required reading before coding

Read these files in order:

1. `README.md`
2. `00-system-architecture.md`
3. `01-etsy-api-notes.md`
4. `02-printer-integration-notes.md`

Do not start coding until you understand the Etsy order/listing flow and the Bambu P1S safety constraints.

## Product goal

Build a local-first web app that helps Hammad run an Etsy 3D-print-on-demand shop:

- manage approved printable products and variants
- create Etsy draft listings from product metadata
- receive/import Etsy paid orders
- map order lines to print jobs
- queue jobs for a Bambu P1S with AMS
- require operator approval before starting prints
- track packing/shipping
- post tracking back to Etsy after labels are purchased outside Etsy API

## Non-negotiable constraints

- Do not store secrets in git.
- Do not auto-print unknown or unmapped orders.
- Do not start a printer job without explicit operator approval in v1.
- Do not claim Etsy can buy labels via API; it cannot per Etsy fulfillment docs.
- Build printer integration behind feature flags and a `PrinterAdapter` interface.
- Draft Etsy listings first; do not auto-publish active listings until review flow exists.

## Suggested tech stack

Use this unless Hammad has an existing repo/stack:

- Node.js + TypeScript
- Fastify or Express API
- React + Vite dashboard
- Prisma + SQLite for MVP
- Zod validation
- Vitest for unit tests
- Playwright optional for UI smoke tests

## Environment variables

Create `.env.example` with placeholders only:

```env
DATABASE_URL=file:./dev.db
APP_BASE_URL=http://localhost:3000

ETSY_CLIENT_ID=
ETSY_CLIENT_SECRET=
ETSY_REDIRECT_URI=http://localhost:3000/api/etsy/oauth/callback
ETSY_SHOP_ID=
ETSY_WEBHOOK_SIGNING_SECRET=

PRINTER_AUTOMATION_ENABLED=false
PRINTER_UPLOAD_ENABLED=false
PRINTER_START_PRINT_ENABLED=false
PRINTER_MODEL=P1S
PRINTER_HOST=
PRINTER_SERIAL=
PRINTER_USERNAME=bblp
PRINTER_ACCESS_CODE=
PRINTER_MQTT_PORT=8883
PRINTER_FTPS_PORT=990
```

## Implementation stages

### Stage 1 — Project scaffold

Objective: Create a runnable monorepo skeleton.

Tasks:

1. Create `apps/api` TypeScript backend.
2. Create `apps/web` React frontend.
3. Create shared package for types if useful.
4. Add root scripts:
   - `dev`
   - `test`
   - `lint`
   - `typecheck`
5. Add `.env.example`.
6. Add README setup instructions.

Verification:

```bash
npm install
npm run typecheck
npm test
npm run dev
```

### Stage 2 — Database schema

Objective: Implement the entities from `00-system-architecture.md`.

Create Prisma models for:

- Product
- ProductVariant
- EtsyListing
- EtsyOrder
- EtsyOrderLine
- PrintJob
- Printer
- Shipment
- OAuthToken or EtsyConnection

Rules:

- Store raw Etsy receipt/transaction JSON on order/order-line rows.
- Store printer access code only as a secret reference or encrypted field if a secret manager is absent. For MVP local dev, use `.env`, not DB.

Tests:

- Can create product + variant.
- Can create Etsy order + line.
- Can create print job from variant.

### Stage 3 — Etsy client

Objective: Centralize Etsy API access.

Implement:

- OAuth authorize URL builder.
- OAuth callback token exchange.
- Token refresh.
- `createDraftListing(product)`.
- `updateListingInventory(listingId, variants)`.
- `getShopReceipts(filters)`.
- `getReceiptTransactions(receiptId)`.
- `postReceiptTracking(receiptId, shipment)`.

Use exact paths/scopes from `01-etsy-api-notes.md`, but verify against the current OpenAPI spec while coding.

Tests:

- Header builder includes Bearer token and x-api-key.
- Request body for draft listing includes required fields.
- Receipt transaction parser maps `sku`, `listing_id`, `product_id`, `quantity`, `variations`.
- Tracking payload uses `tracking_code` and `carrier_name`.

### Stage 4 — Product catalog API/UI

Objective: Hammad can define approved products and variants.

Backend endpoints:

- `GET /api/products`
- `POST /api/products`
- `GET /api/products/:id`
- `POST /api/products/:id/variants`
- `PATCH /api/variants/:id`

Frontend:

- Product list
- Product detail
- Variant editor

Validation:

- SKU required and unique.
- Variant file path required.
- Nozzle defaults to `0.4`.
- Material required.

### Stage 5 — Listing draft publisher

Objective: Create Etsy draft listings from approved products.

Backend:

- `POST /api/products/:id/etsy/draft-listing`
- stores Etsy listing ID and state
- does not activate listing automatically

Frontend:

- Button: “Create Etsy Draft”
- Shows result listing ID/link
- Shows missing data blockers before creating

Tests:

- Missing shipping profile blocks draft creation for physical listing.
- Missing image assets warns operator.
- Successful mock Etsy response creates `EtsyListing` row.

### Stage 6 — Order intake

Objective: Import paid unshipped orders and map to print jobs.

Backend:

- `POST /api/webhooks/etsy` — verify signature, dedupe by webhook ID, enqueue receipt sync.
- `POST /api/etsy/sync-orders` — manual fallback poll.
- Order mapper that converts transaction SKUs to variants.

Rules:

- If SKU maps to variant: create `PrintJob` rows with status `queued`.
- If no mapping: mark line as `needs_manual_mapping`; no print job.
- If duplicate webhook/order: idempotent no-op.

Tests:

- Valid webhook creates/syncs order.
- Duplicate webhook does not duplicate jobs.
- Missing SKU mapping blocks print job.
- Quantity 3 creates either one job with quantity 3 or three jobs; choose one and document rationale.

### Stage 7 — Manual print queue

Objective: Hammad can run production without printer automation yet.

Frontend:

- Print queue list grouped by status.
- Job detail shows SKU, material, colour, file path, order, buyer/order ID, quantity.
- Buttons:
  - mark blocked
  - mark ready
  - mark printed
  - cancel

Backend:

- status transition endpoint with validation
- audit timestamps

Tests:

- Invalid status transitions rejected.
- Printed job updates order-line production state.

### Stage 8 — Printer adapter interface + manual adapter

Objective: Prepare for P1S automation without enabling it.

Implement:

- `PrinterAdapter` interface.
- `ManualPrinterAdapter`.
- `PrinterService` that reads feature flags.

Tests:

- When automation disabled, startPrint returns operator instructions and does not contact printer.
- Start print requires job state `awaiting_approval` or equivalent.

### Stage 9 — Bambu P1S read-only status

Objective: Connect to MQTT and show printer status without sending commands.

Implement:

- MQTT client wrapper for TLS port `8883`.
- Subscribe to `/device/{serial}/report`.
- Persist last raw event and mapped status.

Feature flag:

```env
PRINTER_AUTOMATION_ENABLED=true
PRINTER_START_PRINT_ENABLED=false
```

Tests:

- Unit-test mapper with fixture MQTT payloads captured from actual printer.
- App remains healthy if printer offline.

### Stage 10 — FTPS upload + gated start print

Objective: Controlled printer automation after manual tests pass.

Implement:

- FTPS upload to printer SD card.
- MQTT command builder for `project_file` and/or `gcode_file`.
- Operator approval endpoint:
  - validates safety gates
  - uploads file if needed
  - starts print only if `PRINTER_START_PRINT_ENABLED=true`

Tests:

- Safety gate rejects wrong nozzle, missing file, missing approval, non-idle printer.
- Command builder snapshot tests.
- Manual integration test with Hammad-approved calibration file.

### Stage 11 — Fulfillment/tracking

Objective: Post tracking to Etsy after package is shipped.

Backend:

- `POST /api/orders/:id/shipments`
- `POST /api/shipments/:id/post-to-etsy`

Frontend:

- shipment form: carrier, tracking code, ship date, note to buyer
- button: “Post tracking to Etsy”

Rules:

- Do not mark shipped without tracking unless Hammad explicitly allows.
- Store Etsy response.
- Idempotently handle already-posted shipment.

Tests:

- Tracking payload contains `tracking_code`, `carrier_name`, `send_bcc`.
- Posting tracking updates `posted_to_etsy_at`.

## Definition of done for MVP

- Product/variant catalog works locally.
- Etsy client has mocked tests for all request builders/parsers.
- Manual order sync works with mocked Etsy responses.
- Paid order creates print queue items only for mapped SKUs.
- Printer automation defaults off.
- Manual print queue is usable without printer integration.
- P1S integration is behind feature flags and safety gates.
- Shipment tracking can be posted to Etsy after manual label purchase.
- README clearly explains setup and risks.

## First thing to ask Hammad before implementation

Ask which repo/path to build in and whether this should start as:

1. a fresh monorepo from scratch, or
2. part of an existing app/repo.

Do not create production Etsy listings or start printer jobs during development.
