---
name: Browse the Lemon Perfect catalog
description: >-
  Read Lemon Perfect's product catalog anonymously — search, list, and retrieve
  products and variants — using the Storefront GraphQL API on lemonperfect.com.
  Read-only; no purchase, no buyer data, no credentials required.
api: graphql/lemon-perfect-storefront-2026-07.graphql
endpoint: https://lemonperfect.com/api/2026-07/graphql.json
operations: [products, product, productByHandle, search, predictiveSearch, collections, collection, collectionByHandle, productRecommendations, productTags, productTypes, shop]
generated: '2026-08-04'
method: generated
---

# Browse the Lemon Perfect catalog

Lemon Perfect is a single-brand direct-to-consumer beverage store. The catalog is
small — seven products at the time of writing (Original, Dragon Fruit, Peach,
Blueberry, Watermelon, Coconut, and a 4-Flavor Variety Pack) — so prefer listing
everything over paginating.

## Endpoint and auth

`POST https://lemonperfect.com/api/2026-07/graphql.json` with
`Content-Type: application/json`.

The Storefront API normally expects an `X-Shopify-Storefront-Access-Token`
header. On this store, anonymous reads returned HTTP 200 without one when probed
on 2026-08-04. Send the header if you have a token; do not fabricate one.

The version segment is a calendar version. `2026-07` is current; query
`{ publicApiVersions { handle supported } }` to confirm before pinning.

## Steps

1. **Establish store context.** Query `shop` for `name`, `description`, and
   `paymentSettings { currencyCode countryCode }` so prices are interpreted in
   the right currency (USD/US at time of writing).
2. **List the catalog.** Use `products(first: 50)` and select
   `edges { node { id handle title description availableForSale priceRange { minVariantPrice { amount currencyCode } } } }`.
   Seven products fit in one page — no cursor walk needed.
3. **Search by intent.** Use `search(query: "...", types: [PRODUCT], first: 10)`
   for full text, or `predictiveSearch(query: "...")` for type-ahead.
4. **Fetch one product.** Use `productByHandle(handle: "15-2-original")` when you
   have the URL handle, or `product(id: ...)` with a global object id. Select
   `variants(first: 10) { edges { node { id title availableForSale quantityAvailable price { amount currencyCode } } } }`
   — the **variant id** is what you add to a cart, not the product id.
5. **Browse merchandising.** `collections(first: 20)` and
   `collectionByHandle(handle: "all")` expose the store's own groupings.
   `productRecommendations(productId: ...)` returns related items.

## Conventions to respect

- **Pagination** is Relay cursor connections: `first`/`after` plus
  `pageInfo { hasNextPage endCursor }`. There is no offset paging.
- **Cost, not call count.** Every response carries
  `extensions.cost.requestedQueryCost` and a `shopify-complexity-score-v2`
  header. Ask only for the fields you need; a `THROTTLED` error in `errors[]`
  means back off.
- **Errors arrive on HTTP 200.** Check the top-level `errors[]` array on every
  response — see `errors/lemon-perfect-error-codes.yml`.
- **Deprecated fields** are marked with `@deprecated` and a machine-readable
  reason in the schema (54 of them). Read the reason and use the replacement.
- Log the `x-request-id` response header with any failure you report.

## Do not

- Do not scrape the HTML storefront when this API answers the question — the
  store's own `/agents.md` asks agents to prefer the structured surfaces.
- Do not use this skill to place an order. Purchasing runs through the UCP/MCP
  endpoint and requires explicit human approval: see
  `lemon-perfect-agentic-purchase.md`.
