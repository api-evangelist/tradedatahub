---
name: tradedatahub-enumerate-catalog
description: Enumerate the complete TradeDataHub purchasable dataset catalog deterministically over the free API, instead of scraping thousands of HTML pages.
api: TradeDataHub Public API
generated: '2026-08-29'
method: generated
source: openapi/tradedatahub-openapi.json
operations:
  - GET /api/v1/coverage
  - GET /api/v1/states
  - GET /api/v1/trades
  - GET /api/v1/cities
  - GET /api/v1/datasets
note: >-
  The provider's OpenAPI declares NO operationIds, so operations here are named by HTTP method and
  path exactly as they appear in openapi/tradedatahub-openapi.json. Synthesised operationIds are
  proposed separately in overlays/tradedatahub-openapi-overlay.yaml and are NOT used here.
---

# Enumerate the TradeDataHub catalog

Base URL: `https://www.tradedatahub.net`. Every step is free, unauthenticated and read-only. Do not
scrape `/leads/*` or `/datasets/*` HTML pages — the API returns the same inventory deterministically.

## 1. Start at coverage

```
GET /api/v1/coverage
```

Returns `record_count`, `live_states`, `trades`, `cities`, `product_count`, `currency`, `price_model`
and `last_updated`. Use `product_count` as the expected size of the full catalog and `last_updated` as
the data freshness date.

## 2. Read the dimensions

```
GET /api/v1/states     # product_id, state, record_count, price, available_for_purchase
GET /api/v1/trades     # trade, record_count, product_count
GET /api/v1/cities?state=Texas   # state, city, record_count, trade_count
```

Filter values elsewhere must match these canonical names **exactly and case-sensitively**
(`Texas`, `HVAC Contractor`). An unknown `?state=` on `/cities` returns HTTP 404.

## 3. Page the catalog

```
GET /api/v1/datasets?limit=100&offset=0
```

- `limit` is 1..100 (default 100). A larger value is silently clamped to 100 and still returns 200 —
  do not rely on the declared 400.
- Read `pagination.total`, then increment `offset` by the effective `pagination.limit` until
  `offset >= pagination.total`.
- Optional filters: `state`, `trade`, `city`, `type` (`city_trade|state_trade|state|mega_pack`).
- A filter that matches nothing returns HTTP 200 with `datasets: []` and `pagination.total: 0` — an
  empty result, not an error.

## 4. Respect the envelope

Every response carries `api_version`. Errors arrive as `{api_version, error:{code, message}}` with
`application/json` — this API does **not** use RFC 9457 `application/problem+json`. See
`errors/tradedatahub-problem-types.yml`.

## Pacing

No rate limits are published and no `RateLimit-*` or `Retry-After` headers are returned. Responses are
edge-cached for 60 seconds (`cache-control: public,max-age=60`). Pace conservatively — roughly 86 pages
covers the full catalog at `limit=100`.
