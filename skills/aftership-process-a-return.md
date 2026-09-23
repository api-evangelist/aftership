---
name: aftership-process-a-return
description: Create, approve, receive, resolve or reject a return through the AfterShip Returns API, addressed either by return ID or by RMA number. Use when an agent is handling a merchant's reverse-logistics workflow.
api: AfterShip Returns API
base_url: https://api.aftership.com/returns/2026-07
operations:
  - post-returns
  - get-returns
  - get-returns-return_id
  - get-returns-rma-rma_number
  - post-returns-return_id-approve
  - post-returns-rma-rma_number-approve
  - post-returns-return_id-reject
  - post-returns-return_id-receive-items
  - post-returns-return_id-resolve
  - post-returns-return_id-remove-items
  - post-returns-rma-rma_number-attach-shipments
  - post-returns-rma-rma_number-dropoffs-dropoff_id-drops
  - post-create-return-deep-link
  - get-item-tags
generated: '2026-08-27'
method: generated
source: openapi/aftership-returns-api-openapi.yml + https://www.aftership.com/docs/returns/quickstart/api-quick-start
---

# Process a return (AfterShip Returns API)

Base `https://api.aftership.com/returns/2026-07`, `as-api-key` header, same `{"meta":..., "data":...}`
envelope as every other AfterShip API.

## Pick one addressing mode and stay in it

Almost every action exists **twice** — once keyed on `return_id` and once on `rma_number`:

| Action | By return ID | By RMA number |
|---|---|---|
| Read | `get-returns-return_id` | `get-returns-rma-rma_number` |
| Approve | `post-returns-return_id-approve` | `post-returns-rma-rma_number-approve` |
| Reject | `post-returns-return_id-reject` | `post-returns-rma-rma_number-reject` |
| Resolve | `post-returns-return_id-resolve` | `post-returns-rma-rma_number-resolve` |
| Receive items | `post-returns-return_id-receive-items` | `post-returns-rma-rma_number-receive-items` |
| Update items | `patch-returns-return_id-items-item_id` | `patch-returns-rma-rma_number-items-item_id` |
| Remove items | `post-returns-return_id-remove-items` | `post-returns-rma-rma_number-remove-items` |
| Attach shipments | `post-returns-return_id-attach-shipments` | `post-returns-rma-rma_number-attach-shipments` |

Use the RMA number when you are acting on something a shopper quoted, and the return ID when you are
acting on something you created in this session. Do not mix them within one workflow.

## The happy path

1. `post-returns` (`POST /returns`) to create the return.
2. `post-returns-return_id-approve` — or `-reject` if it fails policy.
3. `post-returns-rma-rma_number-attach-shipments` once the shopper has a return label, so the inbound
   leg shows up in Tracking.
4. `post-returns-rma-rma_number-dropoffs-{dropoff_id}-drops` if the shopper used a drop-off point.
5. `post-returns-return_id-receive-items` at the warehouse.
6. `post-returns-return_id-resolve` to settle the refund or exchange.

## Handing the shopper a portal instead

`post-create-return-deep-link` (`POST /returns/link`) mints a deep link into the AfterShip-hosted
returns page for a specific order. Prefer this over walking a shopper through the API yourself.

## Reversing

There is no "un-approve". The reversals that exist are:

- `post-returns-return_id-remove-items` — the shopper changed their mind about part of the return.
  Mirrors the "Remove Return Items from an Existing RMA" admin action. No time window is stated.
- `post-returns-return_id-reject` — transition to rejected.

No window is published for either, so confirm with the merchant before acting rather than assuming a
grace period exists.

## Rate limit

Flat **10 requests/second per organization** — not the per-endpoint scheme the Tracking API uses.
Headers on 429: `meta.type` `TooManyRequests`.
