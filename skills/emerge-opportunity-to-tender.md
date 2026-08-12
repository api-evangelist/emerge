---
name: Source truckload capacity on Emerge and tender it
description: >-
  Run the Emerge opportunity-to-tender workflow end to end — authenticate, create a freight
  opportunity, post it to the Emerge Marketplace and/or your network partners, receive carrier
  options by webhook, then tender the load to the chosen carrier and follow it to delivery.
api: openapi/emerge-public-api-openapi.yml
base_url: https://api.emergemarket.io/v1
sandbox_url: https://demo-api.emergemarket.dev/v1
generated: '2026-08-12'
method: generated
source: openapi/emerge-public-api-openapi.yml
operations:
  - POST /auth/login
  - POST /opportunities
  - POST /opportunities/{opportunity_id}/post_to_marketplace
  - POST /opportunities/{opportunity_id}/post_to_network_partners
  - GET /opportunities/{opportunity_id}/options
  - POST /tenders
  - GET /tenders/{tender_id}
  - GET /shipments/{shipment_id}
---

# Source truckload capacity on Emerge and tender it

This is the workflow Emerge calls **opportunity-to-tender**: everything from sourcing through
tendering and tracking stays in the Emerge system. Use it when Emerge is the execution surface. If
your own TMS executes the load, use `emerge-opportunity-to-award.md` instead.

> The Emerge OpenAPI documents define **no `operationId` on any operation**, so every step below
> cites the real HTTP method and path from `openapi/emerge-public-api-openapi.yml`. Do not invent
> operation identifiers.

## Before you start

- You need an Emerge shipper account. There is no self-serve API signup.
- Develop against the sandbox `https://demo-api.emergemarket.dev/v1` first. It is a **separate
  registrable domain**, not a key prefix — credentials do not carry environment information, so
  never assume a token is scoped to test.
- Emerge publishes **no test loads, test carriers or magic identifiers**. Anything you create in
  production is real freight.

## 1. Authenticate

`POST /auth/login` with `{"user_name": ..., "password": ...}`. The response carries the access
token; send it as `Authorization: Bearer <token>` on every subsequent call.

- This endpoint is rate limited to **20 requests per second**, and Emerge says the limit may be
  lower during high-volume periods. Cache the token; do not log in per request.
- Refresh with `POST /auth/refresh` rather than re-submitting the password.
- On `401` the body is empty — re-authenticate rather than trying to parse an error.

## 2. Create the opportunity

`POST /opportunities` with the load: `stops` (pickup and delivery with schedule and contact),
`commodities`, `equipment_type_id`, `equipment_size`, `load_type_id`, `accessorials`, `references`
and `notes`. Set `is_hot_load` and `is_tracking_required` where they apply.

**Always send your own `customer_reference_number` in `references`.** Emerge documents **no
idempotency key** on any write operation, so a retried `POST /opportunities` after a timeout will
create a duplicate load. Your reference is the only way to detect and clean that up — and
`DELETE /opportunities/by_customer_reference/{customer_reference_number}` is the only lookup keyed
on it.

Before any retry: `GET /opportunities` and check whether your reference is already there.

## 3. Put it in front of capacity

Two levers, and they can be used together:

- `POST /opportunities/{opportunity_id}/post_to_marketplace` — exposes the load to the Emerge
  Marketplace (45,000+ pre-vetted asset-based carriers).
- `POST /opportunities/{opportunity_id}/post_to_network_partners` — sends it to your own private
  network partners (`GET /network_partners` to see who that is).

Both are externally visible commercial actions. Treat them as human-confirmed steps in an agent
flow. `POST /opportunities/{opportunity_id}/unpost_from_marketplace` withdraws it.

## 4. Collect options

Prefer the webhook: subscribe to `options_changed` and Emerge pushes the option set as it changes
(see `emerge-subscribe-to-events.md`). Whether all options or only the lowest one is sent is an
application setting on the Emerge side, not an API parameter.

Polling fallback: `GET /opportunities/{opportunity_id}/options`. Note there are **no pagination
parameters** on this or any other collection read — you get what you get in one response.

## 5. Tender to the chosen carrier

`POST /tenders` referencing the `opportunity_id` and the `option_id` you selected. This commits the
load to a carrier — an irreversible commercial and physical action. Confirm with a human.

- `GET /tenders/{tender_id}` to read the current status (`tender_status`).
- `PUT /tenders/{tender_id}` to amend.
- `POST /tenders/{tender_id}/cancel` to cancel — returns **409** when the tender is not in a
  cancellable state. Re-read the tender before retrying rather than looping on the 409.

## 6. Follow the shipment

`GET /shipments/{shipment_id}` returns the shipment with its `tenders`, `current_tender` and
`status_id`. For live movement, subscribe to the tracking events — `tracking_status_updated`,
`delivery_status_updated`, `stop_actual_times_updated`, and `current_location_updated_job` (emitted
every 30 minutes) — rather than polling.

## Error handling

Emerge does not use RFC 9457. Errors are a proprietary envelope, and `code` just repeats the HTTP
status, so **the HTTP status is the taxonomy**:

| Status | Body | What to do |
|---|---|---|
| 400 | `{"error":{"code":400,"messages":[...]}}` | Read `messages`; fix the request. Do not retry unchanged. |
| 401 | empty | Re-authenticate. |
| 403 | `{"error":{"code":403,...}}` | Account lacks access — stop, do not retry. |
| 404 | empty | Bad id. |
| 409 | `{"error":{"code":409,...}}` | State conflict (tender cancel). Re-read, then decide. |
| 429 | `{"code":429}` | Back off. **No `Retry-After` or `RateLimit-*` header is returned** — use your own exponential backoff. |
| 500 | `{"code":500}` | Retry with backoff. |

Full catalog: `errors/emerge-problem-types.yml`. Cross-cutting rules:
`conventions/emerge-conventions.yml`.
