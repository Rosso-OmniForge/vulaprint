# POS Slip Receiver App Integration Guide

This guide is for the receiving/printing application that consumes POS slips from backend.

Use this together with POS_SLIP_API.md.

## Goal

Replace all legacy A4/PDF POS invoice printing behavior with queue-based printing from backend APIs.

Your receiver app should:

1. Poll for pending slips.
2. Fetch full payload for each slip.
3. Print the slip.
4. Mark it completed only after successful physical print.
5. Continue polling as source of truth.

## Base URL and Auth

- Base URL example: https://shop.novaconv.co.za
- Header required on all printer APIs: `X-API-Key: <PRINTER_API_KEY>`
- Missing/wrong key returns 401.
- If backend printer key is not configured, returns 503.

### Printer user identity (required)

Every POS slip endpoint also requires the backend user ID of the cashier queue this printer instance should serve. Accepted via **either** transport:

- Header (preferred): `X-Printer-User-Id: <user_id>`
- Query parameter: `?user_id=<user_id>`

Both transports may be sent simultaneously and must carry the same value; a mismatch returns 400. Missing or non-positive-integer values return 400.

Configure your receiver app with the user ID of the assigned cashier account at startup. This ID is how the backend routes slips to the correct printer.

## Endpoints to Implement

1. GET /admin/api/pos-slips/pending
2. GET /admin/api/pos-slips/request/{request_id}
3. POST /admin/api/pos-slips/complete
4. Optional monitoring: GET /admin/api/printer-app/status (admin session auth)

## Recommended Runtime Loop

Use a worker loop every 5-30 seconds.

### Step A: Poll queue

Call GET /admin/api/pos-slips/pending.

- If empty: sleep and retry.
- If non-empty: process in created_at order.

### Step B: Process each request id

For each pending id:

1. Fetch detail payload.
2. Validate minimum printable fields.
3. Render/format to receipt layout.
4. Send to printer.
5. If printer success: mark completed.
6. If failure: do NOT complete; log and retry later.

### Step C: Retry policy

- Network/API failures: exponential backoff (e.g. 1s, 2s, 4s, max 30s).
- Printer failures: keep request pending and retry later.
- Completion endpoint failure after print success: retry completion only.

## Idempotency and Safety Rules

1. Treat backend queue state as authoritative.
2. Never mark complete before confirmed print success.
3. If completion fails after print success, retry completion until success.
4. If request is no longer pending when retrying complete, treat as already resolved.
5. Keep a local short-lived cache of in-flight request ids to avoid duplicate concurrent prints.

## Payload Mapping Checklist

From GET /admin/api/pos-slips/request/{request_id}, print these sections:

1. Header/business block
- business.brand_name
- business.phone
- business.email
- business.vat_number
- business.address_line1
- business.address_line2
- business.city
- business.province
- business.postal_code
- business.country

2. Transaction meta
- request.invoice_number
- request.created_at
- cashier_username
- request.payment_type
- customer_email (optional)

3. Store block
- store.name
- store.address
- store.phone
- store.email

4. Items
- items[].title
- items[].variant_label
- items[].sku
- items[].qty
- items[].unit_price_cents
- items[].line_tax_cents
- items[].line_total_cents

5. Totals
- totals.subtotal_before_discount_cents
- totals.manual_discount_cents
- totals.voucher_discount_cents
- totals.subtotal_cents
- totals.tax_cents
- totals.total_cents
- totals.vat_bps
- totals.currency

6. Footer
- footer_note

## Money and Tax Formatting

- All monetary values are integers in cents.
- Convert cents to display with two decimals (e.g. 259900 -> 2599.00).
- VAT basis points: vat_bps / 100 gives percent value.
  - Example: 1500 bps -> 15.00%

## Connection Status Interpretation

Backend marks receiver app online when API requests with valid key are seen recently.

- GET /admin/api/printer-app/status returns:
  - connected: true/false
  - last_seen_at
  - last_seen_source
  - seconds_since_seen
  - online_window_seconds

Use this for admin diagnostics only; do not use it as queue source of truth.

## Error Handling Matrix

1. GET pending -> 401
- Cause: wrong/missing API key
- Action: stop worker, surface auth error

2. GET pending -> 503
- Cause: backend missing PRINTER_API_KEY
- Action: stop worker, alert ops/admin

3. GET pending -> 400
- Cause: missing, non-integer, or conflicting printer user identity
- Action: stop worker, fix receiver app configuration (PRINTER_USER_ID env var)

4. GET detail -> 404
- Cause: request removed/not found, or request belongs to a different user
- Action: skip id and continue

5. POST complete -> 400 (not pending)
- Cause: already completed/cancelled
- Action: treat as resolved and continue

6. POST complete -> 404
- Cause: request not found or owned by different user
- Action: treat as not applicable and continue

7. Any endpoint -> timeout/network error
- Action: retry with backoff

## Suggested Local Data You Should Track

1. in_flight_ids set
- Prevent same request being printed twice concurrently.

2. print_attempt_count per request
- Useful for alerting repeated failures.

3. last_successful_poll_at
- Health metric for your receiver process.

4. last_successful_print_at
- Operational visibility for support.

## Minimal Pseudocode

```text
loop forever:
  pending = GET /admin/api/pos-slips/pending
  for req in pending ordered by created_at:
    if req.id in in_flight_ids:
      continue
    mark in_flight_ids add req.id
    try:
      detail = GET /admin/api/pos-slips/request/{id}
      printable = map_payload(detail)
      ok = print_to_device(printable)
      if ok:
        POST /admin/api/pos-slips/complete { request_id: id }
      else:
        log print failure, leave pending
    except:
      log/retry policy
    finally:
      remove id from in_flight_ids
  sleep poll_interval
```

## Example HTTP Calls

```bash
curl -sS \
  -H "X-API-Key: $PRINTER_API_KEY" \
  -H "X-Printer-User-Id: $PRINTER_USER_ID" \
  https://shop.novaconv.co.za/admin/api/pos-slips/pending
```

```bash
curl -sS \
  -H "X-API-Key: $PRINTER_API_KEY" \
  -H "X-Printer-User-Id: $PRINTER_USER_ID" \
  https://shop.novaconv.co.za/admin/api/pos-slips/request/42
```

```bash
curl -sS -X POST \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $PRINTER_API_KEY" \
  -H "X-Printer-User-Id: $PRINTER_USER_ID" \
  -d '{"request_id":42}' \
  https://shop.novaconv.co.za/admin/api/pos-slips/complete
```

## Go-Live Checklist

1. Confirm PRINTER_API_KEY is configured in backend env.
2. Confirm PRINTER_USER_ID is configured in receiver app env (the backend user.id of the assigned cashier).
3. Validate 401 path with bad key and 200 path with correct key + valid user ID.
4. Validate 400 path with missing user ID.
5. Print one real POS sale end to end.
6. Confirm request moves pending -> completed only after print success.
7. Confirm a second receiver configured with a different user ID cannot see or complete slips owned by the first.
8. Confirm label printing and POS slip printing can coexist in app worker logic.
9. Add receiver app logs for poll, print, complete, and retry events.

## Support Handoff Notes

When reporting issues, provide:

1. request_id
2. invoice_number
3. receiver app timestamp
4. API response code/body
5. printer error output

This is enough to correlate backend queue state with receiver-side behavior quickly.
