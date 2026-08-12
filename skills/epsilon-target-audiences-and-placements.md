---
name: epsilon-target-audiences-and-placements
description: >-
  Configure who sees which Epsilon Retail Media ads — sync shoppers and segments
  for audience targeting, and set up the filter mappings and cross-sell
  categories that shape placement selection.
api: Epsilon Retail Media Integration API
spec: openapi/epsilon-retail-media-integration-openapi.json
operations:
  - create-or-update-a-customer
  - retrieve-a-list-of-customers
  - retrieve-a-customer
  - manage-customers-and-segments
  - createFilterMapping
  - listFilterMappings
  - getFilterMapping
  - updateFilterMapping
  - deleteFilterMapping
  - listCrossSellCategories
  - createCrossSellCategory
  - getCrossSellCategory
  - deleteCrossSellCategory
generated: '2026-08-12'
method: generated
source: >-
  Grounded in the operationIds present in
  openapi/epsilon-retail-media-integration-openapi.json,
  openapi/epsilon-retail-media-filter-mapping-openapi.json and
  openapi/epsilon-retail-media-cross-sell-category-openapi.json, plus
  https://developers.citrusad.com/integration/reference/audience-targeting-new,
  /filtermapping and /category-cross-sell
---

# Target audiences and shape placements

Three separate APIs, three separate credential shapes. Read this before wiring
targeting, because the auth model changes between them.

## Audience targeting — Integration API (Basic + API key)

Epsilon documents three integration options: curated unified audiences, syncing
customers *and* audiences, or syncing audiences only. If you are syncing
customers:

1. **Sync shoppers** — `create-or-update-a-customer` (`POST /customers`) with a
   `customers` array. Keyed on your own `customerId`.
2. **Link segments** — `manage-customers-and-segments`
   (`POST /customers/manage-segments`) with `teamId`, `customerId` and
   `segments`. Segment membership is managed here, not by rewriting the customer.
3. **Verify** — `retrieve-a-list-of-customers`
   (`GET /customers?limit=&skip=&customerIds=`) and `retrieve-a-customer`
   (`GET /customers/{customerId}`).
4. **Use it** — pass `customerId` and/or `audience` on `POST /ads/generate`.

> This endpoint writes shopper identity data. It is governed by the platform
> Acceptable Use Policy at
> `https://developers.citrusad.com/integration/reference/acceptable-use-policy`.
> Sync only what you are entitled to sync.

## Filter mappings — Filter Mapping API (JWT Bearer)

Maps your own product filter vocabulary onto the filters Epsilon applies during
ad selection, so `productFilters` on an ad request means what you think it means.

- `createFilterMapping` — `POST /filter-mapping`
- `listFilterMappings` — `GET /filter-mapping?limit=&skip=`
- `getFilterMapping` — `GET /filter-mapping/{id}`
- `updateFilterMapping` — `PATCH /filter-mapping/{id}`
- `deleteFilterMapping` — `DELETE /filter-mapping/{id}`

Auth here is `Authorization: Bearer xxx.yyy.zzz` (JWT), not Basic.

> The published spec for this API declares `info.version: 0.0.1-alpha`. Treat it
> as less stable than the rest of the platform.

## Cross-sell categories — Cross-Sell Category API (JWT Bearer)

Defines the category-to-category relationships behind the cross-sell placement,
where ads for one category are served against products in a related category.

- `listCrossSellCategories` — `GET /v2/cross-sell-categories`
- `createCrossSellCategory` — `POST /v2/cross-sell-categories`
- `getCrossSellCategory` — `GET /v2/cross-sell-categories/{id}`
- `deleteCrossSellCategory` — `DELETE /v2/cross-sell-categories/{id}`

There is **no update operation** — change a cross-sell category by deleting and
recreating it.

Then request cross-sell ads with `POST /ads/generate` using a cross-sell
`placement` and `targetedProductGtin`.

## Rules that bite

- **Three auth models across three APIs.** Basic API key on Integration, JWT
  Bearer on Filter Mapping and Cross-Sell Category, OAuth 2.0 bearer on `/ads`
  only. See `authentication/epsilon-authentication.yml`.
- **Three error envelopes too.** `text/plain` on v1, `{"message": …}` on filter
  mapping and cross-sell, RFC 9457 `problem+json` on brand pages. See
  `errors/epsilon-problem-types.yml`.
- **Deletes are unrecoverable and not idempotency-protected.** There is no
  undo and no replay guard. Read with the `get` operation first.
- **Category cross-sell is not the category placement.** They are distinct
  placement types; Epsilon publishes a page explaining the difference.
