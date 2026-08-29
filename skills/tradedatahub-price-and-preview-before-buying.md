---
name: tradedatahub-price-and-preview-before-buying
description: Confirm a TradeDataHub dataset's price and inspect its masked availability preview before committing to a purchase, and inspect the x402 payment challenge without paying.
api: TradeDataHub Public API
generated: '2026-08-29'
method: generated
source: openapi/tradedatahub-openapi.json
operations:
  - GET /api/v1/datasets
  - GET /api/v1/datasets/{product_id}
  - GET /api/v1/datasets/{product_id}/price
  - GET /api/v1/datasets/{product_id}/preview
  - GET /api/v1/datasets/{product_id}/download
note: >-
  The provider's OpenAPI declares NO operationIds; operations are named by method and path as they
  appear in openapi/tradedatahub-openapi.json.
---

# Price and preview a TradeDataHub dataset before buying

Base URL: `https://www.tradedatahub.net`. Steps 1–4 are free and unauthenticated.

## 1. Find the product_id

```
GET /api/v1/datasets?state=Florida&trade=Roofer
```

Always take `product_id` from an API response. The documented grammar is
`state:{state_slug}`, `state-trade:{state}:{trade}`, `city-trade:{state}:{city}:{trade}` and
`mega-pack:seven-live-states`, but the provider explicitly instructs clients not to construct ids
ad hoc.

## 2. Read the authoritative price

```
GET /api/v1/datasets/{product_id}/price
```

Returns `amount_cents`, `price`, `currency`, `record_count`, `last_updated` and
`available_for_purchase`. Treat this endpoint as authoritative over any price on a marketing page or
in `llms.txt`. The response shape is identical to the dataset detail response (`Price` is a `$ref` to
`Dataset` in the contract).

## 3. Inspect the masked preview

```
GET /api/v1/datasets/{product_id}/preview
```

Only `city_trade` and `state_trade` products have previews. Asking for a `state` or `mega_pack`
preview returns `{"error":{"code":"preview_unavailable", ...}}` **with HTTP 200** — branch on
`error.code`, not on the status code.

A preview returns `fields[]` and `records[]` where `business` is always the literal
`"Masked business"` and phone/website are reduced to `phone_available` / `website_available`
booleans. Use it to judge field coverage and `verification_date`, never to extract contact data.

## 4. Inspect the payment challenge without paying

```
GET /api/v1/datasets/{product_id}/download
```

With no `PAYMENT-SIGNATURE` header this returns **HTTP 402** carrying the live x402 v2 challenge both
as a base64 `Payment-Required` response header and as a `payment_required` object in the body. The
provider states agents may inspect this challenge without making any payment.

Read `accepts[0]` for `scheme`, `network`, `amount`, `asset`, `payTo` and `maxTimeoutSeconds`. Never
hardcode any of it — the challenge is generated per request.

## 5. Before you settle — know what you cannot undo

- The rail is **TESTNET ONLY**: Base Sepolia (`eip155:84532`), testnet USDC. Mainnet settlement is not
  enabled, so do not treat a successful x402 retrieval as production commerce.
- **There is no programmatic reversal.** No cancel, refund or void operation exists in the contract.
- The refund path is human and out of band: email `tradedatahub@gmail.com` within **7 days** of
  purchase with the Stripe Checkout session id (prefix `cs_`). See
  <https://www.tradedatahub.net/refunds/>. Change of mind after successful delivery is not eligible.
- A purchased CSV is delivered by a signed link valid **24 hours**, allowing up to **5** downloads.

## 6. Data-use obligations

Per the provider's own terms: inclusion of a business in a dataset is **not** consent to be contacted.
The buyer is responsible for TCPA, CAN-SPAM and state privacy compliance. `last_verified_date`
describes source processing only — not licensing, active business status, or consent.
