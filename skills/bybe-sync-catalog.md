---
name: Sync the BYBE product and manufacturer catalog
description: >-
  Pull BYBE's manufacturer, product and store reference data into a local catalog so offers can be
  matched to basket line items by UPC before a redemption is submitted.
api: openapi/bybe-api-openapi-original.yml
generated: '2026-08-13'
method: generated
source: openapi/bybe-api-openapi-original.yml
operations:
  - 'GET /v1/manufacturers'
  - 'GET /v1/manufacturers/{id}'
  - 'GET /v1/products'
  - 'GET /v1/products/{id}'
  - 'GET /v1/stores'
  - 'GET /v1/stores/{id}'
operation_id_note: >-
  The BYBE v1 specification declares no operationId on any operation. Steps cite METHOD + PATH
  exactly as published; no operationId is invented.
---

# Sync the BYBE product and manufacturer catalog

Base URL `https://api.bybe.io`. HTTP Basic auth (`api_key` : `api_secret`). All read-only — this is
the safe half of the BYBE API and the right place for an agent to start.

## Why sync at all

Redemption submission fails on products BYBE does not know (`422 unknown_upcs`). Holding a local
copy of the catalog lets you decide *before* the money-moving call which basket lines are even
eligible, and lets you show shoppers offers matched to what they actually bought.

## Step 1 — manufacturers

```
GET /v1/manufacturers?page=1&limit=100
GET /v1/manufacturers/{id}
```

Manufacturers are the brand owners whose promotions BYBE distributes. Their integer `id` is the
`manufacturer_id` filter on products.

## Step 2 — products

```
GET /v1/products?page=1&limit=100
GET /v1/products?manufacturer_id=568374289
GET /v1/products?product_type_id=&product_subtype_id=&product_package_type_id=
GET /v1/products/{id}
```

Key the local table on **`upc`** — the specification names it the preferred product identifier. Your
own SKU can only be used as `product.retailer_identifier` at redemption time if you pre-registered it
with BYBE, so UPC is the durable join.

`product_type_id`, `product_subtype_id` and `product_package_type_id` are integer filters; the
specification documents no endpoint that enumerates their values, so discover them by walking the
product list and collecting the distinct ids you see.

## Step 3 — stores

```
GET /v1/stores?state=OH
GET /v1/stores/{id}?page=1&limit=100
```

`state` takes a two-letter US abbreviation from `state_string_enum`. Stores must already have been
registered with BYBE by you; this is how you verify what BYBE currently holds against your own store
list before a redemption references one.

Note the paging quirk in the published contract: `page` and `limit` are declared on
`GET /v1/stores/{id}`, not on `GET /v1/stores`.

## Step 4 — page correctly

`page` and `limit` are the only pagination controls, and **no response declares a total count, a
next-page field, or a `Link` header**. Page until a page comes back shorter than `limit`, then stop.
Do not assume a fixed default page size — the specification does not state one.

## Step 5 — refresh cadence

Catalog data has no published change feed: BYBE ships no webhooks, no AsyncAPI, and no changelog, and
sends no `ETag`/`Last-Modified` you can condition on. A full periodic re-walk is the only mechanism
available. Offers move faster than the catalog — re-read `GET /v1/offers` per session rather than
caching it, because `budget_remaining_cents` and `redemptions_remaining` are live counters.

## Operational notes

- No published rate limits, no `429` in the contract, no `Retry-After`. A full catalog walk is the
  heaviest thing you will do against this API — run it off-peak, serially, with backoff on 5xx.
- Every response carries `x-request-id`; log it (support@bybe.com). Status:
  <https://status.bybe.com/>.
- A `401` means the Basic credential is missing or wrong. It arrives as **HTML, not JSON**, and is
  not declared anywhere in the specification — do not try to parse it as an error envelope.
