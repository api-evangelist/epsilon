---
name: epsilon-onboard-retail-media-catalog
description: >-
  Stand up a retailer on Epsilon Retail Media from an empty sandbox to a catalog
  that can return ads — create the catalog, sync products, verify the sync, and
  confirm ads are being generated.
api: Epsilon Retail Media Integration API
spec: openapi/epsilon-retail-media-integration-openapi.json
operations:
  - create-or-update-a-catalog
  - create-or-update-a-product
  - retrieve-a-list-of-products
  - retrieve-a-product
  - generate
generated: '2026-08-12'
method: generated
source: >-
  Grounded in the operationIds present in
  openapi/epsilon-retail-media-integration-openapi.json and the flow documented at
  https://developers.citrusad.com/integration/docs/sync-your-products and
  https://developers.citrusad.com/integration/reference/syncing-catalog-via-api
---

# Onboard a retailer catalog on Epsilon Retail Media

Ads cannot be returned until a catalog exists, has products in it, and has at
least one active approved campaign. An empty catalog returns **no ads and no
error** — the most common first-run confusion.

## Before you start

- A sandbox created for you by your Epsilon Technical Account Manager. There is
  no self-service signup.
- Your **teamId** and **API key** for the sandbox, from Integration Settings in
  the platform UI. These values are different in production.
- Your assigned base URL. Every documented host is a template; Epsilon assigns
  the real one.

Authenticate every call with:

```
Authorization: Basic base64(<apiKey>:)
```

See `authentication/epsilon-authentication.yml`.

## Steps

1. **Create the catalog** — `create-or-update-a-catalog` (`POST /catalogs`).
   Send a `catalogs` array. For single-catalog namespaces your Technical Account
   Manager may have already created it; creating it again is an upsert.

2. **Sync products** — `create-or-update-a-product` (`POST /catalog-products`).
   Send a `catalogProducts` array. Products are keyed by **GTIN** within the
   catalog, so re-sending the same GTIN updates rather than duplicates. This is
   the only idempotence the API gives you — there is no `Idempotency-Key` header
   anywhere on this platform.

3. **Verify the sync** — `retrieve-a-list-of-products`
   (`GET /catalog-products?catalogId=…&limit=…&skip=…`) and spot-check individual
   items with `retrieve-a-product`
   (`GET /catalog-products/{catalogId}/{gtin}`). Paging is offset-style:
   `limit` + `skip`.

4. **Confirm ads generate** — `generate` (`POST /ads/generate`) with
   `catalogId`, `placement` and `maxNumberOfAds` (all three are required), plus
   `searchTerm` for a search placement. An empty `ads` array here means no active
   approved campaign, not a broken integration.

## Rules that bite

- **`catalogId`, `placement` and `maxNumberOfAds` are required** on
  `/ads/generate`. Omitting `catalogId` returns `400 text/plain` with the body
  `"catalogId must be set"` — a bare string, not JSON. `/ads/bannerx` returns a
  differently-worded message (`"catalog id is empty"`) for the same mistake, so
  do not string-match on error text.
- **Errors are not uniform.** v1 endpoints answer `text/plain`; the v3
  brand-pages surface answers RFC 9457 `application/problem+json`. See
  `errors/epsilon-problem-types.yml`.
- **Hold a persistent HTTP connection.** The docs require it for ad generation;
  a new TLS handshake per request measurably degrades ad response time.
- **Fall back gracefully.** If an ad request times out or errors, render the page
  without sponsored placements. Never fail the shopper's page on it.
- **Back off blind on 429.** It is declared on every operation but there is no
  `Retry-After` and no `RateLimit-*` header. Implement your own exponential
  backoff. See `rate-limits/epsilon-rate-limits.yml`.
