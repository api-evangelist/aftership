---
name: aftership-create-and-track-shipment
description: Register a shipment with AfterShip Tracking, read its checkpoints, predict its delivery date, and recover from the common failure modes. Use when an agent needs live shipment status for a merchant account.
api: AfterShip Tracking API
base_url: https://api.aftership.com/tracking/2026-07
operations:
  - create-tracking
  - get-tracking-by-id
  - get-trackings
  - detect-courier
  - retrack-tracking-by-id
  - mark-tracking-completed-by-id
  - delete-tracking-by-id
  - predict
generated: '2026-08-27'
method: generated
source: openapi/aftership-tracking-api-openapi.yml + https://www.aftership.com/docs/tracking/quickstart/api-quick-start
---

# Create and track a shipment (AfterShip Tracking API)

Every request goes to `https://api.aftership.com/tracking/2026-07` with an `as-api-key` header.
Every response — success or failure — is wrapped in `{"meta": {...}, "data": {...}}`. Branch on
`meta.code`, **not** on the HTTP status: `meta.code` 4004 arrives with HTTP 404, and 4012 with HTTP 400.

## 1. Register the shipment

Call `create-tracking` (`POST /trackings`) with `tracking_number`. Omit `slug` and let AfterShip
auto-detect the carrier unless the merchant told you which one it is.

- `meta.code` **4003** — "Tracking already exists". This is not an error to retry; it means the number
  is already registered. Fall through to step 2 and read it.
- `meta.code` **4011** — the carrier needs extra fields beyond `tracking_number`; the message names them
  (e.g. `destination_postal_code`, `tracking_account_number`). Ask the merchant, then re-submit.
- `meta.code` **4012** — auto-detection failed or the carrier is not on the account's approved list.
  Call `detect-courier` (`POST /couriers/detect`) to see the candidates, or send `slug` explicitly.

There is **no idempotency key** on this API. Duplicate protection comes only from the 4003 uniqueness
check, so never fire `create-tracking` twice hoping the second call is a no-op for a different resource.

## 2. Read status

`get-tracking-by-id` (`GET /trackings/{id}`) for one shipment, `get-trackings` (`GET /trackings`) for a
page. Read `data.tracking.tag` for the coarse status and `subtag` for the specific one — the sub-status
vocabulary is AfterShip's own (`Delivered_005`, `Exception_016`, `InTransit_011`, ...) and is the thing
worth surfacing to a shopper. From 2026-07 the object also carries `shipment_dimensions`,
`proof_of_delivery`, `multi_piece_info`, and `shipment_direction` with `return_shipment` /
`forward_shipment` links between the outbound and return legs.

Deduplicate checkpoints on `checkpoints[].hash` (added in 2026-07) rather than on timestamp.

## 3. Predict a delivery date

`predict` (`POST /estimated-delivery-date/predict`) for one shipment, `predict-batch` for many.
Batch is rate-limited far more generously (20 req/s vs 5 req/s), so prefer it whenever you have more
than one shipment in hand.

## 4. Rate limits

Limits are **per endpoint**, per organization, per second — not one bucket for the API:
`POST /trackings` 20, `GET /trackings` 6, `GET /trackings/{id}` 5, `POST /couriers/detect` 3,
`POST /estimated-delivery-date/predict-batch` 20. On 429 (`meta.type` `TooManyRequests`) back off
exponentially and honour `X-RateLimit-Reset`.

## 5. Reversing what you did

- Registered the wrong shipment → `delete-tracking-by-id` (`DELETE /trackings/{id}`).
- Tracking expired and needs to resume → `retrack-tracking-by-id`. **This is bounded.** The OpenAPI says
  "Max 3 times per tracking"; the error reference returns `meta.code` 4016 "You can only retrack each
  shipment once." AfterShip contradicts itself here — assume **once** and do not burn the attempt
  speculatively. Only an *inactive* tracking can be retracked (`meta.code` 4013 otherwise).
- Shipment is done but the carrier never posted a final event → `mark-tracking-completed-by-id`.

## 6. Prefer webhooks over polling

Do not poll `GET /trackings`. Subscribe a webhook destination and consume `tracking_update`,
`edd_revise` and `tracking_pending_time`. AfterShip retries a failed delivery up to 14 times over
roughly 68 hours with `2^retry × 30s` backoff, and expects a 2xx.
See `asyncapi/aftership-webhooks.yml`.
