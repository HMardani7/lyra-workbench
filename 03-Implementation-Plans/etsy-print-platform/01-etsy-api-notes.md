# Etsy Open API v3 Notes for Kiro

Sources checked:
- Etsy API docs/tutorials via `developers.etsy.com`
- Etsy OpenAPI spec: `https://www.etsy.com/openapi/generated/oas/3.0.0.json`
- Etsy webhooks docs and 2026 event naming note

## Authentication

Etsy Open API v3 uses OAuth2. Authenticated requests need:

```http
Authorization: Bearer <access_token>
x-api-key: <etsy_app_keystring>
```

Some third-party client notes mention `x-api-key: <client_id>:<shared_secret>` for newer Etsy behavior. Kiro must verify the current Etsy docs/account behavior during implementation and centralize this in one `EtsyClient` header builder.

Do **not** commit Etsy secrets or OAuth tokens. Store them in `.env` locally and encrypted/secret storage later.

## MVP scopes

- `listings_r`
- `listings_w`
- `shops_r`
- `shops_w`
- `transactions_r`
- `transactions_w`

## Listing creation

Endpoint from OpenAPI spec:

```http
POST /v3/application/shops/{shop_id}/listings
Content-Type: application/x-www-form-urlencoded
OAuth scope: listings_w
Operation: createDraftListing
```

Required form fields:

```json
{
  "quantity": 10,
  "title": "Example 3D Printed Item",
  "description": "Human-written listing description.",
  "price": 12.99,
  "who_made": "i_did",
  "when_made": "made_to_order",
  "taxonomy_id": 1234
}
```

Common optional fields for physical made-to-order 3D prints:

```json
{
  "shipping_profile_id": 123456,
  "return_policy_id": 123456,
  "readiness_state_id": 123456,
  "materials": ["PLA", "PETG"],
  "tags": ["3d printed", "desk accessory", "gift"],
  "item_weight": 80,
  "item_weight_unit": "g",
  "item_length": 100,
  "item_width": 80,
  "item_height": 40,
  "item_dimensions_unit": "mm",
  "is_supply": false,
  "should_auto_renew": true,
  "is_taxable": true,
  "type": "physical"
}
```

Valid enums from spec:

- `who_made`: `i_did`, `someone_else`, `collective`
- `when_made`: `made_to_order`, `2020_2026`, `2010_2019`, `2007_2009`, `before_2007`, etc.
- `item_weight_unit`: `oz`, `lb`, `g`, `kg`
- `item_dimensions_unit`: `in`, `ft`, `mm`, `cm`, `m`, `yd`, `inches`
- `type`: `physical`, `download`, `both`

For Hammad's print-on-demand shop, default should be:

```json
{
  "who_made": "i_did",
  "when_made": "made_to_order",
  "type": "physical",
  "is_supply": false
}
```

## Listing images

Etsy tutorial says published listings require at least one image.

Endpoint from OpenAPI spec:

```http
POST /v3/application/shops/{shop_id}/listings/{listing_id}/images
Content-Type: multipart/form-data
OAuth scope: listings_w
Operation: uploadListingImage
```

Flow:

1. Create draft listing.
2. Upload at least one listing image with binary file parameter `image`.
3. Update listing state to `active` only after Hammad reviews.

## Inventory and SKUs

Endpoint from OpenAPI spec:

```http
PUT /v3/application/listings/{listing_id}/inventory
Content-Type: application/json
OAuth scope: listings_w
Operation: updateListingInventory
```

Required request body shape:

```json
{
  "products": [
    {
      "sku": "SKU-COLOR-MATERIAL",
      "property_values": [],
      "offerings": [
        {
          "price": 12.99,
          "quantity": 10,
          "is_enabled": true
        }
      ]
    }
  ],
  "price_on_property": [],
  "quantity_on_property": [],
  "sku_on_property": []
}
```

The exact nested product schema should be generated from the OpenAPI spec in code. The important architectural point: **Etsy transaction `sku` should map back to `ProductVariant.sku`** so orders can be converted into print jobs.

## Order intake

Preferred path:

- Etsy webhook `order.paid` posts to our app.
- App verifies signature.
- App fetches receipt using `resource_url` or explicit receipt endpoint.
- App fetches receipt transactions.

Fallback path:

```http
GET /v3/application/shops/{shop_id}/receipts?was_paid=true&was_shipped=false&sort_on=updated&sort_order=desc
OAuth scope: transactions_r
Operation: getShopReceipts
```

Useful query params from spec:

- `min_created`
- `max_created`
- `min_last_modified`
- `max_last_modified`
- `limit`
- `offset`
- `sort_on`: `created`, `updated`, `receipt_id`
- `sort_order`: `asc`, `ascending`, `desc`, `descending`, `up`, `down`
- `was_paid`
- `was_shipped`
- `was_delivered`
- `was_canceled`

## Receipt response fields

`ShopReceipt` fields from OpenAPI spec include:

```json
{
  "receipt_id": 1234567890,
  "receipt_type": 0,
  "seller_user_id": 123,
  "buyer_user_id": 456,
  "buyer_email": "may require approved access for commercial apps",
  "name": "Buyer Name",
  "first_line": "Address line 1",
  "second_line": "Address line 2",
  "city": "City",
  "state": "State/County",
  "zip": "Postal Code",
  "status": "open",
  "formatted_address": "...",
  "country_iso": "GB",
  "payment_method": "...",
  "message_from_buyer": "...",
  "is_paid": true,
  "is_shipped": false,
  "create_timestamp": 1760000000,
  "created_timestamp": 1760000000,
  "update_timestamp": 1760000000,
  "updated_timestamp": 1760000000,
  "is_gift": false,
  "gift_message": "...",
  "grandtotal": {},
  "subtotal": {},
  "total_price": {},
  "total_shipping_cost": {},
  "shipments": [],
  "transactions": [],
  "refunds": []
}
```

Do not rely on buyer email for MVP because Etsy notes commercial apps with `transaction_r` need separate approval for `buyer_email`.

## Receipt transactions

Endpoint:

```http
GET /v3/application/shops/{shop_id}/receipts/{receipt_id}/transactions
OAuth scope: transactions_r
Operation: getShopReceiptTransactionsByReceipt
```

`ShopReceiptTransaction` fields from spec include:

```json
{
  "transaction_id": 987654321,
  "title": "Listing title",
  "description": "Listing description",
  "seller_user_id": 123,
  "buyer_user_id": 456,
  "quantity": 1,
  "listing_image_id": 111,
  "receipt_id": 1234567890,
  "is_digital": false,
  "listing_id": 222,
  "transaction_type": "listing",
  "product_id": 333,
  "sku": "SKU-COLOR-MATERIAL",
  "price": {},
  "shipping_cost": {},
  "variations": [],
  "product_data": [],
  "shipping_profile_id": 444,
  "min_processing_days": 1,
  "max_processing_days": 3,
  "expected_ship_date": 1760000000
}
```

Order mapper rule:

1. Prefer exact `sku` match to `ProductVariant.sku`.
2. Fallback to `listing_id + product_id` mapping.
3. If no mapping, mark order line `needs_manual_mapping` and do not create a print job.

## Webhooks

Webhook docs state Etsy sends HTTP POSTs with headers:

- `webhook-id`
- `webhook-timestamp`
- `webhook-signature`

Common payload fields:

```json
{
  "event_type": "order.paid",
  "resource_url": "https://api.etsy.com/v3/application/shops/12345678/receipts/987654321",
  "shop_id": 12345678
}
```

As of 2026 Etsy notes lowercase dot-separated event names; use `order.paid`, not old `ORDER_PAID`.

Webhook verification design:

```text
signed_content = webhook-id + "." + webhook-timestamp + "." + raw_body
expected = HMAC-SHA256(signed_content, decoded_signing_secret)
compare expected base64 digest with signature header value after `v1,`
reject if timestamp too old
reject duplicates by webhook-id
```

Kiro must implement verification against Etsy's current webhook docs and unit-test with known fixture values.

## Fulfillment/tracking

Endpoint:

```http
POST /v3/application/shops/{shop_id}/receipts/{receipt_id}/tracking
Content-Type: application/json
OAuth scope: transactions_w
Operation: createReceiptShipment
```

Minimal useful body:

```json
{
  "tracking_code": "TRACKING123",
  "carrier_name": "Royal Mail",
  "send_bcc": true,
  "note_to_buyer": "Thanks for your order — your 3D print has shipped."
}
```

Additional optional fields from spec include `mail_class`, package dimensions/weight, `ship_date`, countries, customs data, label cost/currency, etc.

Important limitation from Etsy fulfillment tutorial:

- Etsy Open API v3 does **not** provide an endpoint to purchase shipping labels.
- The platform should store label/tracking data after Hammad buys the label manually or through another shipping integration.
