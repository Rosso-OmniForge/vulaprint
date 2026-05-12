# POS Slip Printing API (Backend)

This document describes the backend API contract for POS slip printing integration.

## Overview

POS slips are queued automatically when a POS order is submitted.
The external print app can:

1. Poll for pending slip requests.
2. Fetch full payload details for each request.
3. Mark requests as completed after printing.

The backend also supports an optional best-effort webhook signal when a new slip request is created.

Legacy POS A4/PDF invoice endpoints have been removed. POS slip printing now flows only through this printer-app API contract.

## Authentication

All POS slip endpoints require **both** an API key and a printer user identity.

### API key

- Header: `X-API-Key: <PRINTER_API_KEY>`

If the key is not configured on the backend, endpoints return `503`.
If the key is missing or invalid, endpoints return `401`.

### Printer user identity

Every POS slip endpoint also requires the `user_id` of the backend user whose slip queue this printer instance serves. This is the user that logged the POS sale (cashier). Accepts **either** transport; both are accepted simultaneously and must agree.

- Header: `X-Printer-User-Id: <user_id>` (preferred)
- Query parameter: `?user_id=<user_id>`

If both are present they must carry the same integer value; a mismatch returns `400`.
If neither is present, or the value is not a positive integer, endpoints return `400`.

The identity is used to scope all queue reads and ownership checks:
- Pending slips returned are only those with `target_user_id == <user_id>`.
- Detail and completion endpoints return `404` for any request not owned by the caller's `user_id`.

## Endpoints

### 0) Printer app connection status (admin visibility)

- Method: `GET`
- Path: `/admin/api/printer-app/status`
- Auth: admin session/cookie (not API key)

Response example:

```json
{
  "connected": true,
  "last_seen_at": "2026-05-12T09:15:41.884320Z",
  "last_seen_source": "/admin/api/pos-slips/pending",
  "seconds_since_seen": 8,
  "online_window_seconds": 120
}
```

Notes:

- `connected` is derived from recent successful printer API activity.
- Every successful request using `X-API-Key` updates heartbeat time.
- `online_window_seconds` is configurable by `PRINTER_APP_ONLINE_WINDOW_SECONDS` (default `120`).

### 1) Get pending POS slips

- Method: `GET`
- Path: `/admin/api/pos-slips/pending`
- Requires: `X-API-Key` + printer user identity
- Filters: returns only slips with `target_user_id` matching the caller identity and `status == pending`

Response example:

```json
[
  {
    "id": 42,
    "status": "pending",
    "source": "pos_submit",
    "pos_order_id": 1024,
    "invoice_number": "INV-001024",
    "created_by_username": "cashier1",
    "total_items": 3,
    "total_qty": 5,
    "total_cents": 259900,
    "payment_type": "card",
    "created_at": "2026-05-11T09:40:15.123456Z"
  }
]
```

### 2) Get POS slip request detail

- Method: `GET`
- Path: `/admin/api/pos-slips/request/{request_id}`
- Requires: `X-API-Key` + printer user identity
- Authorization: returns `404` if the request does not exist or `target_user_id` does not match the caller identity (no enumeration leakage)

Response example:

```json
{
  "request": {
    "id": 42,
    "status": "pending",
    "source": "pos_submit",
    "pos_order_id": 1024,
    "invoice_number": "INV-001024",
    "created_by_username": "cashier1",
    "total_items": 3,
    "total_qty": 5,
    "total_cents": 259900,
    "payment_type": "card",
    "created_at": "2026-05-11T09:40:15.123456Z"
  },
  "business": {
    "brand_name": "NovaConv",
    "phone": "+27 11 000 0000",
    "email": "info@novaconv.co.za",
    "vat_number": "4123456789",
    "address_line1": "1 Main Road",
    "address_line2": "Unit 2",
    "city": "Johannesburg",
    "province": "Gauteng",
    "postal_code": "2000",
    "country": "ZA"
  },
  "store": {
    "name": "Sandton Store",
    "address": "123 Street\nSandton\nGauteng\n2196\nZA",
    "phone": "+27 11 111 1111",
    "email": "sandton@novaconv.co.za"
  },
  "totals": {
    "vat_bps": 1500,
    "tax_cents": 33813,
    "subtotal_before_discount_cents": 225900,
    "manual_discount_cents": 10000,
    "voucher_discount_cents": 0,
    "subtotal_cents": 215900,
    "total_cents": 249713,
    "currency": "ZAR"
  },
  "cashier_username": "cashier1",
  "customer_email": "",
  "footer_note": "Thank you for shopping with us.",
  "items": [
    {
      "id": 7001,
      "sku": "SKU-ABC-1",
      "title": "Product A",
      "variant_label": "Black M",
      "unit_price_cents": 79900,
      "qty": 2,
      "line_tax_cents": 23970,
      "line_total_cents": 159800,
      "currency": "ZAR"
    }
  ]
}
```

### 3) Mark POS slip request completed

- Method: `POST`
- Path: `/admin/api/pos-slips/complete`
- Requires: `X-API-Key` + printer user identity
- Authorization: returns `404` if the request does not exist or `target_user_id` does not match the caller identity
- Body:

```json
{
  "request_id": 42
}
```

Response:

```json
{
  "success": true,
  "message": "POS slip print request marked as completed"
}
```

## Optional direct push signal

When `POS_SLIP_WEBHOOK_URL` is set in backend environment variables, backend sends a best-effort POST after POS submission:

- Method: `POST`
- Target: `POS_SLIP_WEBHOOK_URL`
- Headers:
  - `Content-Type: application/json`
  - `X-API-Key: <PRINTER_API_KEY>`
- Body example:

```json
{
  "event": "pos_slip_created",
  "request_id": 42,
  "detail_path": "/admin/api/pos-slips/request/42"
}
```

Notes:

- This push is non-blocking and does not fail checkout if unreachable.
- Polling endpoints remain the source of truth.

## Full POS Slip Payload Contract

### Request envelope (`request`)

- `id` integer: queue request id.
- `status` string: `pending`, `completed`, `cancelled`.
- `source` string: currently `pos_submit`.
- `pos_order_id` integer|null: linked POS order id.
- `invoice_number` string: POS invoice number.
- `created_by_username` string: cashier username at submit time.
- `total_items` integer: unique lines in this slip.
- `total_qty` integer: summed quantity across lines.
- `total_cents` integer: final payable total in cents.
- `payment_type` string: `card`, `cash`, etc.
- `created_at` ISO datetime (UTC).

### Business block (`business`)

- `brand_name`
- `phone`
- `email`
- `vat_number`
- `address_line1`
- `address_line2`
- `city`
- `province`
- `postal_code`
- `country`

All values are snapshotted from Admin Settings -> Business Information at POS submit time.

### Store block (`store`)

- `name`
- `address` (newline-delimited string)
- `phone`
- `email`

All values are snapshotted from the POS order's store data at submit time.

### Totals block (`totals`)

- `vat_bps` integer (e.g. `1500` = 15.00%)
- `tax_cents`
- `subtotal_before_discount_cents`
- `manual_discount_cents`
- `voucher_discount_cents`
- `subtotal_cents`
- `total_cents`
- `currency` (currently `ZAR`)

### Additional top-level fields

- `cashier_username`
- `customer_email`
- `footer_note` (from `pos_slip_footer_note` setting)

### Item rows (`items[]`)

- `id` integer: queue item id.
- `sku` string.
- `title` string.
- `variant_label` string.
- `unit_price_cents` integer.
- `qty` integer.
- `line_tax_cents` integer.
- `line_total_cents` integer.
- `currency` string.

## Printer App Responsibilities

1. Authenticate every request with `X-API-Key`.
2. Poll `/admin/api/pos-slips/pending` at a fixed interval (recommended 5-30 seconds).
3. For each pending id, call `/admin/api/pos-slips/request/{request_id}` and render/print exactly from payload values.
4. After successful physical print, call `/admin/api/pos-slips/complete` once.
5. Treat polling as source of truth even when webhook events are enabled.
6. Handle retries safely:
  - If complete call fails, retry with backoff.
  - If request is no longer `pending`, skip it.
7. Use integer cent values for money calculations/formatting to avoid float rounding drift.
8. Show/record print failures locally without mutating backend state until print succeeds.

## Request lifecycle

- `pending`: created automatically on POS submit.
- `completed`: set by print app after successful print.
- `cancelled`: reserved for future/manual cancellation flows.

## Data source notes

- Business fields come from Admin Settings > Business Information.
- Store fields come from the POS order store snapshot.
- VAT and totals come from finalized POS order values.
- Item values are snapshotted at submission time.
