# Etsy Print Farm Automation — Architecture & Overview

> **For Hermes:** Use subagent-driven-development skill to implement this plan task-by-task.

**Goal:** Build a lights-out Etsy-to-printer pipeline — orders come in from Etsy, the system matches them to pre-sliced GCODE files, sends them to a Bambu Lab P1S printer, and tracks orders/profit with zero human touch between "order received" and "print finished."

**Architecture:** Python CLI + daemon. Polls Etsy API v3 for new orders, resolves listing property values (e.g., "22 inch" variation) to pre-sliced GCODE files on disk, queues the print via FTPS to a Bambu Lab P1S, monitors progress via MQTT, and logs everything to a local SQLite database for metrics.

**Tech Stack:** Python 3.11+, SQLite (SQLModel), requests (Etsy API), paho-mqtt (printer monitoring), bambu-cli (printer control), Typer (CLI), pytest

---

## What This Upgrade Does

### The Problem It Solves

A 3D print Etsy store generates orders at random hours. Without automation, every order requires:
1. Check Etsy for new orders
2. Open the order, read the size/variation selected
3. Navigate to the correct folder on disk, find the right GCODE file
4. Open Bambu Studio, load the file, send to printer
5. Wait for print to finish, remove from bed, start next
6. Track revenue, costs, and profit manually

This is unsustainable beyond a handful of orders. Missed orders, wrong file selection, idle printer time, and lost financial visibility kill margins.

### What This Upgrade Changes

- **New order arrives on Etsy** → system detects it within the polling interval
- System reads the order's listing ID and property values (e.g., "Size: 22 inch")
- System maps `listing_id + property_values` → exact GCODE file path on disk
- System queues the file to the Bambu Lab P1S via FTPS and starts the print
- System monitors the print via MQTT and updates status in the database
- **When print finishes** → notification, next queued job starts (manual bed clearing assumed)
- Dashboard shows: orders, revenue, print time, material cost estimates, profit

### Risk It Prevents

- **Wrong file printed:** Manual file selection risk eliminated — mapping is deterministic from Etsy's property values
- **Missed orders:** Polling catches every order; idempotent so re-runs are safe
- **Printer idle:** Queue drains automatically; printer only stops when queue is empty
- **Lost financial data:** Every order logged with fees, revenue, estimated costs

### Scope

- **Does:** Poll Etsy for new orders, map to GCODE files, send to Bambu Lab P1S, monitor print, track orders + revenue in SQLite
- **Does not:** Handle slicing (files must be pre-sliced), handle Etsy listing creation, handle shipping/fulfillment marking, handle multi-printer farms (v2), handle bed clearing (manual step retained)

---

## System Architecture

```
┌─────────────┐     ┌──────────────┐     ┌───────────────┐     ┌──────────────┐
│  Etsy API   │────▶│ Order Poller  │────▶│ Product Mapper│────▶│ Print Queue  │
│  (v3 REST)  │     │ (every N min) │     │ (folder→file) │     │ (SQLite FIFO)│
└─────────────┘     └──────────────┘     └───────────────┘     └──────┬───────┘
                                                                      │
                                                              ┌───────▼───────┐
                                                              │ Printer Bridge│
                                                              │ FTPS + MQTT   │
                                                              └───────┬───────┘
                                                                      │
                                                              ┌───────▼───────┐
                                                              │ Bambu Lab P1S │
                                                              └───────────────┘
```

### Data Flow

1. **Etsy Poller** (`etsy_store/poller.py`) calls `GET /shops/{id}/receipts` with a `min_created` cursor (last poll timestamp), fetches new receipts with their transactions
2. **Order Processor** (`etsy_store/processor.py`) extracts `listing_id` and property values from each transaction, resolves the matching GCODE file
3. **Product Mapper** (`etsy_store/mapper.py`) consults a mapping config (`products.yaml`) to turn `listing_id + {"Size": "22 inch"}` → `/data/products/iron_man_helmet/helmet_circumference_22inch.gcode.3mf`
4. **Print Queue** (`etsy_store/queue.py`) inserts the job into SQLite with status `pending`
5. **Print Dispatcher** (`etsy_store/dispatcher.py`) picks the next `pending` job, uploads to printer via FTPS, starts print, updates status to `printing`
6. **Print Monitor** (`etsy_store/monitor.py`) listens to MQTT for print progress/completion, updates job status, triggers next dispatch
7. **Metrics** (`etsy_store/metrics.py`) queries the database for orders, revenue, costs, print time, profit

### Product Folder Structure

```
/data/products/
├── iron_man_helmet/
│   ├── helmet_circumference_22inch.gcode.3mf
│   ├── helmet_circumference_23inch.gcode.3mf
│   └── helmet_circumference_24inch.gcode.3mf
├── mjolnir_hammer/
│   ├── full_size_standard.gcode.3mf
│   └── mini_desk_version.gcode.3mf
├── dragon_skull_decor/
│   └── dragon_skull_one_size.gcode.3mf
└── products.yaml     ← mapping config
```

### Database Schema

```
orders
├── id (PK)
├── etsy_receipt_id (unique)
├── etsy_transaction_id
├── listing_id
├── product_name (resolved from mapping)
├── property_values (JSON: {"Size": "22 inch"})
├── gcode_file_path
├── status (pending|printing|done|failed)
├── revenue_amount (cents)
├── etsy_fee_amount (cents)
├── material_cost_estimate (cents)
├── print_time_seconds
├── created_at
└── updated_at
```

---

## Etsy API v3 Response Shapes (Critical Reference)

These are the exact JSON shapes returned by the Etsy API. The implementation must handle these structures.

### Receipt List Response

`GET /v3/application/shops/{shop_id}/receipts?min_created=X&max_created=Y&limit=100&offset=0`

```json
{
  "count": 2,
  "results": [
    {
      "receipt_id": 123456789,
      "receipt_type": 0,
      "seller_user_id": 987654321,
      "buyer_user_id": 111222333,
      "creation_tsz": 1717000000,
      "was_paid": true,
      "was_shipped": false,
      "grandtotal": {
        "amount": 4999,
        "divisor": 100,
        "currency_code": "USD"
      },
      "total_price": {
        "amount": 3999,
        "divisor": 100,
        "currency_code": "USD"
      },
      "total_shipping_cost": {
        "amount": 1000,
        "divisor": 100,
        "currency_code": "USD"
      }
    }
  ]
}
```

### Single Receipt Detail (WITH transactions)

`GET /v3/application/shops/{shop_id}/receipts/{receipt_id}`

```json
{
  "receipt_id": 123456789,
  "grandtotal": {"amount": 4999, "divisor": 100, "currency_code": "USD"},
  "transactions": [
    {
      "transaction_id": 5000001,
      "listing_id": 2001,
      "title": "Iron Man Helmet - 3D Printed Cosplay Prop",
      "quantity": 1,
      "price": {
        "amount": 3999,
        "divisor": 100,
        "currency_code": "USD"
      },
      "variations": [
        {
          "property_id": 513,
          "value_id": 1234,
          "formatted_name": "Size",
          "formatted_value": "22 inch"
        }
      ],
      "product_id": 8000001,
      "sku": "IMH-22"
    }
  ]
}
```

**Key detail:** `transactions[].variations[]` is where customer selections live. Each variation has `formatted_name` (the property, like "Size") and `formatted_value` (the selection, like "22 inch"). The system extracts these into a dict: `{"Size": "22 inch"}`.

For listings WITHOUT variations, `variations` is an empty array `[]`.

### Listing Inventory (Variation Definitions)

`GET /v3/application/listings/{listing_id}/inventory`

```json
{
  "count": 1,
  "results": {
    "listing_id": 2001,
    "products": [
      {
        "product_id": 8000001,
        "sku": "IMH-22",
        "property_values": [
          {
            "property_id": 513,
            "property_name": "Size",
            "value_ids": [1234],
            "values": ["22 inch"]
          }
        ],
        "offerings": [
          {
            "offering_id": 9000001,
            "quantity": 5,
            "price": {"amount": 3999, "divisor": 100, "currency_code": "USD"}
          }
        ]
      }
    ]
  }
}
```

### How the Mapper Uses This

```
Etsy Order (receipt)
  └─ transaction.variations = [{"formatted_name": "Size", "formatted_value": "22 inch"}]
       │
       ▼
  ProductMapper.resolve(
      listing_id=2001,
      property_values={"Size": "22 inch"}   ← extracted from variations
  )
       │
       ▼
  products.yaml lookup:
    listings.2001.property = "Size"
    listings.2001.files["22 inch"] = "helmet_circumference_22inch.gcode.3mf"
       │
       ▼
  /data/products/iron_man_helmet/helmet_circumference_22inch.gcode.3mf
```
