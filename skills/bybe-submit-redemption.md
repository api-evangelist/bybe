---
name: Submit purchases to BYBE and pay a shopper back
description: >-
  Send post-purchase transaction data to BYBE for redemption validation against clipped offers, and
  trigger the consumer cash-back disbursement. This is the money-moving flow of the BYBE API v1.
api: openapi/bybe-api-openapi-original.yml
generated: '2026-08-13'
method: generated
source: openapi/bybe-api-openapi-original.yml
operations:
  - 'POST /v1/redemption_disbursements'
  - 'GET /v1/redemption_disbursements'
  - 'GET /v1/redemption_disbursements/{id}'
  - 'GET /v1/stores'
operation_id_note: >-
  The BYBE v1 specification declares no operationId on any operation. Steps cite METHOD + PATH
  exactly as published; no operationId is invented.
---

# Submit purchases to BYBE and pay a shopper back

Base URL `https://api.bybe.io`. HTTP Basic auth (`api_key` : `api_secret`) on every request.

> **This flow moves money.** A successful call causes BYBE to validate redemptions and email the
> consumer a cash-out. Treat `POST /v1/redemption_disbursements` as a high-consequence action:
> require an explicit human decision before an agent calls it, never call it speculatively, and
> never call it in a retry loop without the deduplication rules in step 4.

## Prerequisites

Three things must already be true, or the call returns `422`:

1. The **consumer** exists — created via `POST /v1/consumers` or implicitly by `POST /v1/clips`
   (see the *Clip a BYBE offer* skill).
2. The **store** was previously registered with BYBE. The specification is explicit: store
   information "must match to a store sent to BYBE previously and will be used for validating
   redemptions." Check with `GET /v1/stores?state=OH` and `GET /v1/stores/{id}`.
3. The **products** are known to BYBE by `upc`, or your SKU was pre-registered as
   `product.retailer_identifier`. UPC is the preferred identifier.

## Step 1 — build the disbursement

```
POST /v1/redemption_disbursements
{"redemption_disbursement": {
  "retailer_identifier": "<your_unique_disbursement_id>",
  "payment_method": "payout_flow",
  "email": "shopper@example.com",
  "consumer": {"retailer_identifier": "<your_loyalty_id>"},
  "purchases": [
    {
      "retailer_identifier": "<your_transaction_id>",
      "purchase_date": "2026-08-12T08:00:00-04:00",
      "store": {"retailer_identifier": "<your_store_id>"},
      "redemptions": [{"offer_id": 980190962}],
      "line_items": [
        {"product": {"upc": "0000000000000"}, "quantity": 1, "price": 1299, "product_name": "..."}
      ]
    }
  ]
}}
```

Field rules that come straight from BYBE:

- `retailer_identifier`, `consumer` and `purchases` are **required**.
- `payment_method` defaults to `payout_flow` and that is the recommended value — it lets the consumer
  accrue a balance and choose how to be paid. Other values vary by retailer; confirm with your BYBE
  representative before sending one.
- `email` may be omitted if the consumer's email was already supplied via the Consumers or Clips API.
- `purchase_date` is ISO 8601, **preferably in the store's timezone** (e.g.
  `"2023-10-01T08:00:00-04:00"`). It defaults to now if omitted — send it explicitly for
  next-day batch submissions or you will misdate every redemption.
- `price` is in **cents**. Money is always split `*_cents` + `*_currency` in this API.
- To redeem the same offer more than once against one purchase, repeat the entry in `redemptions[]`.

## Step 2 — decide your policy on unknown UPCs

```
POST /v1/redemption_disbursements?ignore_unknown_upcs=true
```

- Without the flag (or `false`), a line item whose UPC BYBE does not recognise fails the **whole**
  request with `422` and `{"redemption_disbursement": {"errors": {"unknown_upcs": ["Unknown upcs ..."]}}}`.
- With `ignore_unknown_upcs=true`, unknown items are skipped and the rest of the basket processes.

A mixed retail basket almost always contains non-alcohol items BYBE has never seen, so
`ignore_unknown_upcs=true` is usually correct — but log what was skipped, because a silently skipped
line item is an unpaid rebate.

**Unknown retailer identifiers are different.** Per the specification they **always** return `422`
(`unknown_retailer_identifiers`) and cannot be suppressed by this flag. That error means your store
or consumer registration is out of sync — fix the data, do not retry.

## Step 3 — read the result

`201` on success. Errors use the Rails-style envelope
`{"redemption_disbursement": {"errors": {"<field>": ["<message>"]}}}` — no RFC 9457 problem details,
no error code. Match on the field name.

## Step 4 — retry safely

There is **no `Idempotency-Key` header**. Your `redemption_disbursement.retailer_identifier` and each
`purchase.retailer_identifier` are the deduplication keys, and BYBE's guidance for the equivalent
SFTP path is that the transaction identifier "must be unique for all retailer transactions. If they
are not unique in your system, then you can combine fields to make them unique" — for example
transaction id + purchase date.

So: derive both identifiers deterministically from your own transaction record, **never** from a
random value or a timestamp taken at call time. Then a retry after a timeout carries the same key
and cannot double-pay a shopper.

Confirm before retrying anyway:

```
GET /v1/redemption_disbursements?consumer_retailer_identifier=<your_loyalty_id>
GET /v1/redemption_disbursements/{id}
```

`page` and `limit` paginate; there is no total count in the response.

## Alternative: SFTP batch

If you cannot call the real-time API, BYBE accepts the same post-purchase data as CSV over SFTP
(<https://bybe.com/developers#sftp>), authenticated by an SSH key you provide to BYBE (preferred) or
credentials BYBE provides. One row per line item, with the columns `purchase_date`,
`purchased_product_id`, the consumer identifier, `consumer_email`, `store_identifier`,
`transaction_id`, quantity, price and product name.

## Operational notes

- No published rate limits and no `429` in the contract. Batch, do not hammer.
- Log `x-request-id` from every response — with money in flight it is your only support handle
  (support@bybe.com). Status: <https://status.bybe.com/>.
