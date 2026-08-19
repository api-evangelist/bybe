---
name: Clip a BYBE offer for a shopper
description: >-
  Show live, state-legal alcohol rebate offers inside a retailer app and save ("clip") one for a
  named shopper, safely and idempotently, using the BYBE API v1.
api: openapi/bybe-api-openapi-original.yml
generated: '2026-08-13'
method: generated
source: openapi/bybe-api-openapi-original.yml
operations:
  - 'GET /v1/offers'
  - 'GET /v1/offers/{id}'
  - 'POST /v1/consumers'
  - 'GET /v1/consumers/{retailer_identifier}'
  - 'POST /v1/clips'
  - 'GET /v1/clips'
operation_id_note: >-
  The BYBE v1 specification declares no operationId on any operation. Every step below cites the
  METHOD + PATH exactly as published; no operationId is invented.
---

# Clip a BYBE offer for a shopper

Base URL `https://api.bybe.io`. Every path already carries the `/v1` prefix.

## Authentication

HTTP Basic on every request. Username is your **API key (token)**, password is your **API secret** —
both issued on your BYBE developer page at <https://developer.bybe.io/>.

```
Authorization: Basic base64(api_key:api_secret)
```

There is no OAuth, no scopes, and no test-mode key prefix: the same credential shape is used in
production (`api.bybe.io`) and staging (`api.bybestaging.io`). Never call this API from a browser or
an untrusted client — Basic credentials cannot be scoped down.

## Step 1 — find offers this shopper is legally allowed to see

```
GET /v1/offers?state=OH&consumer_retailer_identifier=<your_loyalty_id>
```

- `state` takes a two-letter US abbreviation from the specification's `state_string_enum` (50 states,
  DC and the US territories). **Always pass it.** Alcohol promotion legality is state-by-state and
  this parameter is how that boundary is expressed in the contract.
- `consumer_retailer_identifier` adds consumer-specific information to the response, such as whether
  this shopper already clipped the offer.
- `prelive=true` includes offers that are not yet live — use it for merchandising previews only,
  never to render a clippable offer to a shopper.
- `show_stores` defaults to `true`; set it `false` when you only need offer metadata and want a
  smaller payload. `show_details=true` adds store detail. `show_clips=true` requires
  `consumer_retailer_identifier`.

Read the budget before you render: an offer carries `budget_remaining_cents` and
`redemptions_remaining` alongside `per_consumer_limit`. An offer whose budget is exhausted is still
returned; do not show it as clippable.

For one offer, `GET /v1/offers/{id}`.

## Step 2 — make sure the shopper exists in BYBE

The shopper is identified **only** by your own loyalty identifier — BYBE never needs your customer's
real identity.

```
POST /v1/consumers
{"consumer": {"retailer_identifier": "<your_loyalty_id>", "payout_email_override": "..."}}
```

Check first with `GET /v1/consumers/{retailer_identifier}` if you want to avoid a duplicate write.

**Pick `retailer_identifier` carefully and never change it.** BYBE's own developer documentation
warns that a consumer identifier "cannot be changed without the consumer losing their accumulated
balance."

## Step 3 — clip the offer

```
POST /v1/clips
{"clip": {
   "offer_id": 980190962,
   "retailer_identifier": "<your_unique_clip_id>",
   "consumer": {"retailer_identifier": "<your_loyalty_id>", "email": "shopper@example.com"}
}}
```

`clip.retailer_identifier` is **your** unique key for this clip, and it is what makes this call safe
to retry.

## Step 4 — handle the response correctly

This is the part agents get wrong. BYBE uses the caller-supplied key for deduplication instead of an
`Idempotency-Key` header:

| Status | Meaning | What to do |
|---|---|---|
| `201` | Clip created | Done. Store the returned `clip.id`. |
| `303` | A clip already exists for this **same offer and consumer** | **Treat as success.** The body carries `location` (e.g. `/v1/clips/980190962`). Follow it; do not retry. |
| `409` | A **different** clip already holds this `retailer_identifier` | Your key collided. Regenerate a genuinely unique key — do **not** retry the same one. |
| `422` | Required field missing (e.g. `retailer_identifier` "can't be blank") | Fix the body. Both `offer_id` and `retailer_identifier` are required. |

Errors are **not** RFC 9457 problem details. The envelope is
`{"clip": {"errors": {"<field>": ["<message>"]}}}` with no machine-readable code, so match on the
field name, then on the message string.

A network timeout on `POST /v1/clips` is safe to retry **with the same `retailer_identifier`** — you
will get a `303` pointing at the clip that was actually created.

## Step 5 — verify

```
GET /v1/clips?offer_id=980190962&consumer_retailer_identifier=<your_loyalty_id>
```

`page` and `limit` paginate. Note that BYBE returns no total count and no `Link` header, so page
until you get a short page.

## Operational notes

- **No rate limits are published.** No `X-RateLimit-*`, no `RateLimit-*`, no `Retry-After`, and no
  documented `429`. Throttle yourself conservatively and back off exponentially on any 5xx.
- Every response carries `x-request-id`. **Log it.** It is the only correlation handle you have when
  contacting BYBE support (support@bybe.com).
- Status: <https://status.bybe.com/>.
