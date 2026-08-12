---
name: Source capacity on Emerge and award it back into your own TMS
description: >-
  Run the Emerge opportunity-to-award workflow — use Emerge purely as a capacity source: create an
  opportunity, post it, take the options, create the award, and execute the load in your own TMS.
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
  - GET /opportunities/{opportunity_id}/options
  - POST /awards
  - GET /awards
  - GET /awards/{id}
  - DELETE /opportunities/by_customer_reference/{customer_reference_number}
---

# Source capacity on Emerge and award it back into your own TMS

Emerge's **opportunity-to-award** workflow is for shippers whose TMS is the system of record for
execution. Emerge sources the capacity and produces the award; your TMS does the tendering,
dispatch and tracking. The dividing line is the award: after `POST /awards`, Emerge is done.

> Neither Emerge OpenAPI defines `operationId`, so steps cite HTTP method + path verbatim from
> `openapi/emerge-public-api-openapi.yml`.

## 1. Authenticate

`POST /auth/login` with the shipper `user_name` and `password`; use the returned token as
`Authorization: Bearer <token>`. Refresh via `POST /auth/refresh`. The login endpoint is capped at
20 requests per second.

## 2. Create the opportunity, carrying your TMS identifier

`POST /opportunities`. Put your TMS load id in `references` as the `customer_reference_number`.

This matters more here than in the tender workflow, because the award has to be matched back into
your TMS. It is also your only duplicate guard: **Emerge documents no idempotency key**, so a
timed-out create must be reconciled by reference, not retried blindly. If you do end up with a
duplicate, `DELETE /opportunities/by_customer_reference/{customer_reference_number}` removes it.

## 3. Post it

`POST /opportunities/{opportunity_id}/post_to_marketplace` for the Emerge Marketplace, and/or
`POST /opportunities/{opportunity_id}/post_to_network_partners` for your own carriers. Withdraw with
`POST /opportunities/{opportunity_id}/unpost_from_marketplace`.

## 4. Take the options

Subscribe to `options_changed` (see `emerge-subscribe-to-events.md`), or poll
`GET /opportunities/{opportunity_id}/options`. No pagination parameters exist on this read.

Each option carries `partner_id`, `rate`, `rate_pre_mile` and `availability_date` — enough to score
against your own contract rates before choosing.

## 5. Award

`POST /awards` with the chosen option. This is the commercial commitment; confirm with a human.

- `GET /awards` and `GET /awards/{id}` to read back.
- Subscribe to `opportunity_awarded` — its payload is explicitly designed so "the subscriber can do
  the subsequent actions, including tendering to it, in their own TMS". That event is the handoff.
- Subscribe to `opportunity_not_awarded` too: it fires when the bid duration expires with no award,
  which is the signal to fall back to your own carrier network.

## 6. Execute in your TMS

Take the award payload into your TMS and tender there. You do **not** call `POST /tenders` in this
workflow — that is the opportunity-to-tender path.

## Reconciliation checklist

- Match on your `customer_reference_number`, never on the Emerge `id` alone.
- `opportunity_deleted` fires if the load is removed on the Emerge side — handle it or your TMS will
  wait forever.
- Watch `network_partner_updated` / `network_partner_deleted` to keep your carrier master in sync.

Error semantics, rate limits and the full event list: `errors/emerge-problem-types.yml`,
`rate-limits/emerge-rate-limits.yml`, `asyncapi/emerge-webhooks.yml`.
