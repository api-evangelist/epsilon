---
name: epsilon-serve-and-attribute-ads
description: >-
  Run the full Epsilon Retail Media revenue loop — request ads for a placement,
  page through them without repeats, report the resulting order, and verify the
  sale attributed.
api: Epsilon Retail Media Integration API
spec: openapi/epsilon-retail-media-integration-openapi.json
operations:
  - generate
  - bannerx
  - report-an-order
  - retrieve-a-list-of-orders
  - retrieve-an-order
generated: '2026-08-12'
method: generated
source: >-
  Grounded in the operationIds present in
  openapi/epsilon-retail-media-integration-openapi.json and the flows documented at
  https://developers.citrusad.com/integration/reference/requesting-product-ads-1,
  /pagination, /reporting-impressions-clicks and /syncing-order-data-via-api
---

# Serve ads and attribute the sale

This is the loop Epsilon Retail Media is paid on: request ads, report
interactions, report the order. If the order never gets reported, the ad spend
never attributes and the retailer never earns on it.

## The one field that matters most

`sessionId` is a value **you** generate per shopper session. It must be
**identical** on the ad request and on the order you later report. Attribution
joins on it. Everything else in this skill is recoverable; a mismatched
`sessionId` silently loses the revenue.

## Steps

1. **Request ads** — `generate` (`POST /ads/generate`).
   Required: `catalogId`, `placement`, `maxNumberOfAds`.
   Placement-specific: `searchTerm` for search placements,
   `targetedProductGtin` for cross-sell placements.
   Optional: `customerId`, `sessionId`, `productFilters`, `options`,
   `bannerSlots`, `audience`, `contentStandardId` (required for banners).

2. **Or request responsive banners** — `bannerx` (`POST /ads/bannerx`), same
   shape minus `targetedProductGtin` and `maxNumberOfAds`.

3. **Page without repeats.** The `generate` response carries a `memoryToken` —
   an opaque base64 continuation encoding the ads already served plus a TTL. Send
   it back on the next request for that session so page 2 does not repeat page 1.
   Product ads only; not supported for banner or Banner X.

4. **Fire tracking.** Impressions and clicks are reported from the URLs in the ad
   response, either client-to-server through your reverse proxy or
   server-to-server to your regional tracking host. See
   `https://developers.citrusad.com/integration/reference/reporting-impressions-clicks`.

5. **Report the order** — `report-an-order` (`POST /orders`) with an `orders`
   array carrying the same `sessionId` and the purchased GTINs.
   **Note: `/orders` must use HTTP Basic with the secret API key.** OAuth 2.0
   bearer tokens work only on `/ads` endpoints.

6. **Verify** — `retrieve-a-list-of-orders`
   (`GET /orders?teamId=…&limit=…&skip=…&orderIds=…`) or
   `retrieve-an-order` (`GET /orders/{orderId}`).

## Rules that bite

- **`POST /orders` has no idempotency protection.** There is no
  `Idempotency-Key` header on this platform. A naive retry after a timeout can
  double-report an order into attribution and billing. Confirm with
  `retrieve-an-order` before retrying, and treat that read as the safe path.
- **Two auth models, one API.** Bearer tokens from `/v1/oauth2/token` are
  accepted on `/ads` only. Everything else needs Basic + API key.
- **Ad generation is in the render path.** Time it out aggressively and fall back
  to an unsponsored page rather than blocking the shopper.
- **Do not put `ads` in your own endpoint path.** Epsilon explicitly advises
  against it because ad blockers pattern-match on it and it costs you revenue.
- **EU retailers:** send the DSA request attribute to receive advertiser and
  ad-financer legal-entity identity in the ad response. See
  `https://developers.citrusad.com/integration/reference/digital-services-act`.
