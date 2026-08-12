---
name: Quote and accept Emerge freight as a Capacity Provider
description: >-
  Operate the Emerge Carrier API as a carrier, broker or TMS vendor — authenticate with client
  credentials, receive rate_request and tender_request events, and respond with a quote or a tender
  response through Capacity Link.
api: openapi/emerge-carrier-api-openapi.yml
base_url: https://api.emergemarket.io/v2
sandbox_url: https://demo-api.emergemarket.dev/v2
generated: '2026-08-12'
method: generated
source: openapi/emerge-carrier-api-openapi.yml
operations:
  - POST /auth/login/client_credentials
  - POST /options
  - POST /tenders/{shipment_id}/responses
---

# Quote and accept Emerge freight as a Capacity Provider

The Carrier API is **event-driven and inverted**: Emerge calls you, and you answer on three
endpoints. There is nothing to poll and nothing to browse — a Capacity Provider cannot list open
freight through this API.

> The Carrier OpenAPI defines no `operationId`; steps cite HTTP method + path from
> `openapi/emerge-carrier-api-openapi.yml`. Note the spec's own version inconsistency: the
> introduction text says "the current version of the API is v1.0.0" while `info.version` is `2.0.0`
> and the servers are `/v2`. **Use `/v2`.**

## 0. Get onboarded

There is no self-serve path. Submit the integration request form linked from the Carrier API docs
(`https://emergetech.zendesk.com/hc/en-us/requests/new?ticket_form_id=11470751569179`). Emerge
issues:

- a **client id / client secret** pair, and
- **`relationship_identifiers`** — the values that will arrive on every `rate_request` so you can
  match the shipper to the right account in your own system. Store the mapping at onboarding; you
  cannot derive it later.

## 1. Authenticate

`POST /auth/login/client_credentials` with the client id and secret. Send the returned token as
`Authorization: Bearer <token>`.

This is a client-credentials exchange at a proprietary path — **not** an RFC 6749 token endpoint.
No scopes exist, so the token is all-or-nothing across the Carrier API. Protect it accordingly and
cache it rather than re-authenticating per event.

## 2. Receive `rate_request`

Emerge sends a `Rate_Request_Event` webhook containing the opportunity details plus the
`relationship_identifiers` object. Steps:

1. Match `relationship_identifiers` to the shipper account in your system.
2. Review the load — stops, commodities, equipment, special requirements, references.
3. Decide whether to price it.

**Capture the `event_id`.** Your response is invalid without it.

## 3. Respond with a quote — or an honest decline

`POST /options` with:

- the `event_id` from the rate request (required), and
- **either** a rate (with duration/validity details) **or** an error reason.

Returns **202**. Two things worth doing properly:

- Put your own quote identifier in `provider_reference`. It is the only handle you get back on the
  resulting Option.
- **Decline explicitly** rather than staying silent. If nothing arrives inside the window configured
  for your Integration Provider profile, Emerge degrades to a manual rate request in the platform
  and emails your team — which is exactly the manual work the integration exists to remove. An error
  response also tells the shipper *why* you did not quote.

Your quote becomes an **Option** in the shipper's view. Emerge's vocabulary: shippers post
*Opportunities*, you return *Quotes*, shippers see *Options*.

## 4. Receive `tender_request` and respond

When the shipper tenders the load, Emerge sends a `Tender_Request_Event`. Answer with
`POST /tenders/{shipment_id}/responses` (also **202**). This accepts or rejects the physical
movement — the most consequential call in this API. In an agent flow, gate it on a human or on a
hard-coded policy (lane, equipment, rate floor, capacity check), never on a model judgement alone.

## Error handling

Same proprietary envelope as the Shipper API, and on the Carrier API it is the flat shape
`{"code": <http status>}` for most responses:

| Status | What to do |
|---|---|
| 400 | Malformed quote/tender response — most often a missing or stale `event_id`. Fix; do not retry unchanged. |
| 401 | Token expired — re-run `POST /auth/login/client_credentials`. |
| 403 | Your provider account is not permitted — stop and contact Emerge. |
| 409 | State conflict; the request no longer applies. |
| 500 | Retry with backoff. |

No `Retry-After` or `RateLimit-*` headers are documented anywhere in this API, and no idempotency
key exists — if a `POST /options` times out, check your own `provider_reference` bookkeeping before
resubmitting.

Reference: `errors/emerge-problem-types.yml`, `conventions/emerge-conventions.yml`,
`asyncapi/emerge-webhooks.yml`.
