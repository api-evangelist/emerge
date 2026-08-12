---
name: Subscribe to Emerge webhook events
description: >-
  Register, verify and operate an Emerge webhook subscription — pick event types, secure the
  callback, survive the retry schedule, and handle each of the 12 shipper events.
api: openapi/emerge-public-api-openapi.yml
base_url: https://api.emergemarket.io/v1
generated: '2026-08-12'
method: generated
source: openapi/emerge-public-api-openapi.yml, asyncapi/emerge-webhooks.yml
operations:
  - POST /auth/login
  - POST /webhooks
  - GET /webhooks
  - GET /webhooks/{webhook_id}
  - PUT /webhooks/{webhook_id}
  - DELETE /webhooks/{webhook_id}
---

# Subscribe to Emerge webhook events

Emerge pushes state changes rather than expecting you to poll — and since **no collection read in
the API is paginated**, webhooks are the only scalable way to stay current. Emerge publishes no
AsyncAPI document; the event catalog lives in the OpenAPI tag groups, and is captured in
`asyncapi/emerge-webhooks.yml`.

## 1. Stand up the receiver first

Before registering, have an HTTPS endpoint that:

- Accepts `POST` with `Content-Type: application/json`.
- Enforces **HTTP Basic** with a username/password you control. This is the *only* authentication
  Emerge offers on delivery — **there is no HMAC signature header**, so Basic credentials plus TLS
  are your entire verification story. Do not skip `is_enabled: true`.
- Returns 2xx fast and processes asynchronously. Slow handlers burn retry attempts.
- Is idempotent on its own side. Emerge retries, and there is no delivery id contract guaranteeing
  exactly-once.

## 2. Register the subscription

`POST /webhooks` with a `webhook_object`:

```
url            your HTTPS receiver
content_type   application/json
is_enabled     true
event_types    [ ... ]        # from the webhook_event_types enum, or ["*"] for everything
authentication { is_enabled: true, username: ..., password: ... }
```

The password is stored in cleartext in the request/response shape. Treat the whole webhook object as
a secret: never log it, never echo it into a support ticket, rotate it with
`PUT /webhooks/{webhook_id}`.

`GET /webhooks` and `GET /webhooks/{webhook_id}` read back; `DELETE /webhooks/{webhook_id}` removes.

## 3. Choose event types deliberately

Subscribing to `"*"` is easy and expensive. The 12 shipper events:

**Tracking (shared envelope, `event_data` varies; `old` state is `{}` on creation)**
- `tracking_status_updated` — primary shipment status, tender accepted → delivered.
- `delivery_status_updated` — on-time status flips (on-time → late), driven by ETA or actual times.
- `stop_actual_times_updated` — actual arrival/departure received.
- `current_location_updated_job` — truck position, emitted **every 30 minutes** per tracked load.
  This is the high-volume one; only subscribe if you render live position.

**Procurement**
- `options_changed` — the option set for an opportunity changed.
- `current_marketplace_option` — marketplace option details changed.
- `opportunity_awarded` — carries the awarded option so you can act in your own TMS.
- `opportunity_not_awarded` — bid duration ended with no award.
- `opportunity_deleted` — opportunity removed on the Emerge side.
- `tender_status_updated` — tender status changed.

**Network**
- `network_partner_updated` — a partner accepted your invite or changed details.
- `network_partner_deleted` — a partner was removed.

Carrier-side integrations receive `rate_request` and `tender_request` on the Carrier API instead —
see `emerge-carrier-quote-and-tender.md`.

## 4. Understand the retry schedule

If your endpoint fails on a network error, server error, or timeout, Emerge retries **6 times**,
each gap measured from the previous attempt:

| Attempt | Gap |
|---|---|
| 1 | 1 second from the initial attempt |
| 2 | 4 seconds |
| 3 | 9 seconds |
| 4 | 16 seconds |
| 5 | 25 seconds |
| 6 | 36 seconds |

Total window is roughly 91 seconds. **After the sixth attempt the event is gone** — there is no
replay endpoint and no event-history read. If your receiver is down for more than ~90 seconds you
must reconcile by polling `GET /opportunities`, `GET /opportunities/{opportunity_id}/options`,
`GET /tenders/{tender_id}` and `GET /shipments/{shipment_id}`. Build that reconciliation path before
you go live; you will need it.

## 5. Operate it

- Alert on your own 5xx rate to the Emerge callback — you have no visibility into their delivery
  logs.
- Use `is_enabled: false` via `PUT /webhooks/{webhook_id}` for planned maintenance rather than
  letting deliveries fail, and reconcile afterwards.
- Emerge's platform status is at https://status.emergemarket.com/ (Shipper API, Carrier API and API
  are separately reported components).
