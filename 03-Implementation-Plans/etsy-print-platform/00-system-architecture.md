# Etsy Print Platform — System Architecture

## What This Platform Does

### The problem it solves

Selling 3D prints on Etsy creates a workflow gap:

1. Product files live locally or in Drive/GitHub.
2. Listings need titles, photos, variations, processing times, and shipping profiles.
3. Orders arrive in Etsy.
4. Each order must be mapped to the correct printable file, colour/material, quantity, and packaging workflow.
5. The printer must have the right filament and should not be started blindly.
6. Tracking must be posted back to Etsy after shipping.

Without a platform, this becomes manual copy/paste across Etsy, slicer, Bambu Studio, printer, and shipping label tools.

### What this platform changes

The platform becomes the operational control center:

- **Product catalog:** Approved printable products with SKUs, Etsy listing data, file paths, supported materials/colours, print time, filament grams, and packaging notes.
- **Listing publisher:** Creates Etsy draft listings from product templates; uploads images; sets inventory/variations; lets Hammad review before activation.
- **Order intake:** Receives `order.paid` webhook events, then fetches the receipt and transactions from Etsy.
- **Production queue:** Converts each order line into one or more print jobs.
- **Printer integration:** Uploads approved `.3mf`/`.gcode.3mf` jobs to the P1S SD card via FTPS and starts/monitors prints over local MQTT only after safety checks.
- **Fulfillment tracker:** Stores packing status, label/tracking details, then posts shipment tracking back to Etsy.

### Scope for v1

**Does:**
- Manage approved products and SKUs.
- Create Etsy draft listings.
- Import paid Etsy orders.
- Map orders to print jobs.
- Track print queue state.
- Integrate with Bambu P1S locally via FTPS/MQTT.
- Require manual operator approval before starting prints.
- Post tracking to Etsy after tracking details exist.

**Does not:**
- Auto-source STLs.
- Auto-slice arbitrary STL files.
- Auto-purchase shipping labels through Etsy. Etsy Open API does not provide label purchase endpoints.
- Let customers upload arbitrary models.
- Run print jobs without a known SKU/file mapping.
- Replace Bambu Studio for complex slicing profiles in v1.

## High-level data flow

```text
Approved product files + metadata
        ↓
Product Catalog DB
        ↓
Listing Publisher ──→ Etsy Draft Listing ──→ Hammad reviews/activates
        ↓
Etsy order.paid webhook / fallback poller
        ↓
Fetch receipt + transactions from Etsy API
        ↓
Order Mapper: transaction SKU/variation → ProductVariant
        ↓
Production Queue: pending_print → ready_to_print → printing → printed
        ↓
Bambu P1S local adapter: FTPS upload + MQTT start/status
        ↓
Pack + buy label manually / shipping tool
        ↓
Post tracking to Etsy receipt
        ↓
fulfilled
```

## Proposed modules

### 1. API server

Responsibilities:
- OAuth token storage/refresh for Etsy.
- Etsy webhook endpoint with signature verification.
- REST API for frontend.
- Background jobs for polling, reconciliation, and printer monitoring.

Recommended stack for Kiro unless Hammad chooses otherwise:
- Node.js + TypeScript
- Fastify or Express
- Prisma + SQLite for MVP; Postgres later if needed
- Zod for API/input validation
- BullMQ/Redis optional later; start with a simple DB-backed job queue for local MVP

### 2. Frontend dashboard

Views:
- Products
- Listings
- Orders
- Print Queue
- Printer Status
- Fulfillment
- Settings

MVP can be a simple React/Vite dashboard.

### 3. Database

Core tables/entities:

```text
Product
- id
- sku_base
- title
- description
- etsy_taxonomy_id
- materials[]
- tags[]
- default_price
- default_quantity
- print_file_id
- packaging_profile_id
- active

ProductVariant
- id
- product_id
- sku
- color
- material
- nozzle_mm
- file_path
- estimated_print_minutes
- estimated_filament_g
- etsy_product_id nullable
- etsy_offering_id nullable

EtsyListing
- id
- etsy_listing_id
- product_id
- state: draft|active|inactive|sold_out
- shipping_profile_id
- readiness_state_id nullable
- last_synced_at

EtsyOrder
- id
- receipt_id
- buyer_display_name
- status
- is_paid
- is_shipped
- created_timestamp
- raw_receipt_json

EtsyOrderLine
- id
- order_id
- transaction_id
- listing_id
- product_id nullable
- variant_id nullable
- sku
- title
- quantity
- selected_variations_json
- raw_transaction_json

PrintJob
- id
- order_line_id nullable
- variant_id
- printer_id
- status: queued|blocked|ready|awaiting_approval|uploading|printing|printed|failed|cancelled
- file_path
- remote_path nullable
- quantity
- started_at nullable
- completed_at nullable
- failure_reason nullable

Printer
- id
- name
- model: P1S
- host
- serial
- access_code_secret_ref
- mqtt_port default 8883
- ftps_port default 990
- enabled

Shipment
- id
- order_id
- carrier_name
- tracking_code
- ship_date
- posted_to_etsy_at nullable
```

## Etsy API integration summary

Use OAuth2 with these scopes for MVP:

- `listings_r` — read listings/inventory
- `listings_w` — create/update listings and inventory
- `shops_r` — read shop/shipping profiles
- `shops_w` — create shipping profiles/readiness states if needed
- `transactions_r` — read receipts/orders/transactions
- `transactions_w` — post tracking to receipts

Important endpoints:

- `POST /v3/application/shops/{shop_id}/listings` — create draft listing.
- `POST /v3/application/shops/{shop_id}/listings/{listing_id}/images` — upload listing image (`listings_w`).
- `PUT /v3/application/listings/{listing_id}/inventory` — set SKU/variation inventory.
- `GET /v3/application/shops/{shop_id}/receipts?was_paid=true&was_shipped=false` — fallback order poller.
- `GET /v3/application/shops/{shop_id}/receipts/{receipt_id}/transactions` — line items.
- `POST /v3/application/shops/{shop_id}/receipts/{receipt_id}/tracking` — mark/order update with tracking.

## Printer integration summary

For Bambu P1S local mode, community docs/libraries show:

- FTPS file upload to printer SD card:
  - host: printer IP
  - port: `990`
  - username: `bblp`
  - password: LAN access code from printer
- MQTT status/control:
  - host: printer IP
  - port: `8883`
  - TLS enabled
  - username: `bblp`
  - password: LAN access code
  - report topic pattern: `/device/{serial}/report`
  - request/control topic pattern commonly `/device/{serial}/request`

Kiro should encapsulate this behind a `PrinterAdapter` interface so the app can run in “manual printer mode” if local printer control is unreliable.

## Safety gates

Before a job can start:

- Product variant has a known file path.
- File exists locally.
- File extension is approved: `.3mf`, `.gcode.3mf`, or `.gcode` depending on tested workflow.
- Nozzle requirement matches printer config, initially `0.4mm`.
- Material/colour requirement is visible to operator; do not assume AMS slot mapping unless it has been configured.
- Printer status is idle.
- Operator clicks “Approve & Start Print.”

## Design decisions

- **Draft listing first:** Avoid accidentally publishing low-quality listings before images/shipping/variants are correct.
- **Manual print approval for v1:** Prevents bad auto-prints, wrong filament, and unattended printer risks.
- **Local printer adapter behind interface:** Bambu local APIs are unofficial/community-documented; app must degrade gracefully.
- **DB-backed queue:** Easier to recover after restarts than in-memory jobs.
- **Store raw Etsy JSON:** Makes debugging API mapping easier without repeatedly hitting rate limits.

## Key risks

- Etsy app access/scopes may be limited until app is configured and approved.
- Etsy webhooks may require approved/commercial app access; build fallback polling.
- Bambu local API is not a formal public API; commands must be tested with the actual P1S.
- Etsy label purchasing is not available via Open API; tracking update is possible but label purchase remains external.
