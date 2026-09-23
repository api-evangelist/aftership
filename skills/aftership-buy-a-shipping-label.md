---
name: aftership-buy-a-shipping-label
description: Rate-shop across carriers, buy a shipping label, manifest it, book a pickup, and cancel any of those through the AfterShip Shipping (Postmen) API. Use when an agent needs to produce outbound shipping documents.
api: AfterShip Shipping API
base_url: https://api.aftership.com/postmen/v3
sandbox_url: https://sandbox-api.aftership.com/postmen/v3
operations:
  - post-shipper-accounts
  - get-shipper-accounts
  - post-rates
  - get-rates
  - post-labels
  - get-label
  - post-cancel-labels
  - post-manifests
  - post-pickups
  - post-cancel-pickups
  - post-address-validations
generated: '2026-08-27'
method: generated
source: openapi/aftership-labels-api-openapi.yml, openapi/aftership-rates-api-openapi.yml, openapi/aftership-cancel-labels-api-openapi.yml + https://www.aftership.com/docs/shipping/quickstart/api-quick-start
---

# Buy a shipping label (AfterShip Shipping / Postmen API)

**Develop against the sandbox first.** This is the only AfterShip product with a published sandbox host,
and every Shipping spec lists it as `servers[0]`:

```
https://sandbox-api.aftership.com/postmen/v3     # sandbox
https://api.aftership.com/postmen/v3             # production
```

Same `as-api-key` header, same `{"meta":..., "data":...}` envelope.

## 1. Connect a carrier

`post-shipper-accounts` (`POST /shipper-accounts`) stores the merchant's carrier credentials.
FedEx has its own onboarding path: `post-v3-couriers-fedex-shipper-accounts`. Read what is already
connected with `get-shipper-accounts` before creating a duplicate.

If you would rather not handle carrier credentials at all, embed the hosted carrier-account element
(`@aftership/carrier-account-sdk`, see `components/aftership-components.yml`).

## 2. Validate the destination

`post-address-validations` (`POST /address-validations`) — beta, but cheaper than a returned parcel.
The standalone Address API (`https://api.aftership.com/address/2024-07/`, `post-addresses/validate`)
does the same job outside a shipping context.

## 3. Rate-shop

`post-rates` (`POST /rates`) returns priced service levels across every connected shipper account.
Read the result back with `get-rate`. This is the closest thing AfterShip offers to a dry run —
it prices the shipment without committing to anything.

## 4. Buy

`post-labels` (`POST /labels`) creates the label. `get-label` retrieves it.

## 5. Hand off

`post-manifests` (`POST /manifests`) closes out the day's labels with the carrier.
`post-pickups` (`POST /pickups`) books collection.

## 6. Reversing

| Bought | Reverse with | Window |
|---|---|---|
| Label | `post-cancel-labels` | none stated by AfterShip — carrier-dependent |
| Pickup | `post-cancel-pickups` | none stated |

Read the cancellation back with `get-cancel-label` / `get-cancel-pickup`; the cancel request is itself
a resource with a status, not a synchronous void. Because no window is published, cancel as soon as you
know rather than assuming you have until end of day.

## Rate limit

Flat **10 requests/second per organization**. The Shipping API signals with lowercase
`rateLimit-limit` / `rateLimit-remaining` / `rateLimit-reset` — **not** the `X-RateLimit-*` headers the
Tracking API uses. One parser will not serve both.
