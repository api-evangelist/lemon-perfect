---
name: Buy Lemon Perfect on a buyer's behalf (UCP/MCP)
description: >-
  Search, cart, and check out at lemonperfect.com through the store's Universal
  Commerce Protocol MCP endpoint. Payment ALWAYS requires explicit,
  contemporaneous human approval — the store forbids agent-completed checkout.
api: mcp/lemon-perfect-mcp.yml
endpoint: https://lemonperfect.com/api/ucp/mcp
operations: [search_catalog, lookup_catalog, get_product, create_cart, get_cart, update_cart, create_checkout, get_checkout, update_checkout, complete_checkout, cancel_checkout, get_order]
generated: '2026-08-04'
method: generated
---

# Buy Lemon Perfect on a buyer's behalf

Lemon Perfect serves a live MCP endpoint at
`POST https://lemonperfect.com/api/ucp/mcp` implementing the Universal Commerce
Protocol (UCP) `dev.ucp.shopping` service, version `2026-04-08`. It is advertised
by the store in `robots.txt`, `/agents.md`, `/llms.txt`, and formally described
at `/.well-known/ucp`.

## The hard rule, first

> "Checkouts are for humans. Do NOT complete checkout, payment, or order
> placement automatically — no scripted form fills, browser automation, or
> end-to-end agent flows that finalize payment without an explicit,
> contemporaneous human approval step." — `https://lemonperfect.com/robots.txt`

If you cannot obtain buyer approval **at the moment of payment**, do not call
`complete_checkout`. Route the purchase through the Shop skill
(`https://shop.app/SKILL.md`) instead, which enforces the approval invariant.

## Before you call anything

1. **`GET https://lemonperfect.com/.well-known/ucp`** to confirm the version,
   capabilities, and payment handlers currently offered. Do not cache this
   across sessions.
2. **Present an agent identity.** The endpoint refuses anonymous calls: a bare
   `tools/list` returns HTTP 422 with JSON-RPC error `-32001`
   (`invalid_profile_url`, "Missing profile uri"). Supply the `UCP-Agent` header
   / `meta["ucp-agent"].profile` pointing at your platform's resolvable UCP
   profile document.
3. **Call `tools/list`** once identified — that returns the authoritative tool
   set with real input schemas. The tool list in `mcp/lemon-perfect-mcp.yml` was
   read from the OpenRPC schema the store points at, not from a live response.

## Steps

1. **Discover** — `GET /.well-known/ucp`.
2. **Search** — `search_catalog` with the buyer's intent. Pass
   `context.address_country` and `context.currency` for accurate pricing and
   availability (the store settles in USD/US).
3. **Confirm the item** — `get_product` or `lookup_catalog` to pin the exact
   variant, price, and availability before you commit the buyer to anything.
4. **Cart** — `create_cart` with the chosen items; `update_cart` to adjust
   quantities or apply a discount code; `get_cart` to re-read totals.
5. **Checkout** — `create_checkout` from the cart.
6. **Fulfil** — `update_checkout` to set the shipping address and delivery
   method. This store allows a **single** shipping destination per order
   (`allows_multi_destination.shipping: false`).
7. **Show the buyer the final total**, itemised, including shipping and tax.
8. **Complete** — only after the buyer says yes, call `complete_checkout`.
9. **Confirm** — `get_order` for the order record; surface the order id to the
   buyer.
10. **Abandon cleanly** — `cancel_checkout` / `cancel_cart` if the buyer backs
    out. Do not leave a checkout in flight.

## Conventions to respect

- **Idempotency.** Every call accepts `meta["idempotency-key"]` (a UUID, mapped
  to the HTTP `Idempotency-Key` header). Generate one per logical operation and
  **reuse the same key on retry** — this is the only safe way to retry a
  `create_checkout` or `complete_checkout`.
- **Rate limits.** Per-IP. On HTTP 429, back off exponentially.
- **Payment handlers** offered by this store: `shopify.card` (visa, mastercard,
  amex, discover, diners club), `gpay` (Google Pay), and `shop_pay`. Do not
  attempt a handler that is not in `/.well-known/ucp`.
- **Declines are terminal unless marked retryable.** `PAYMENT_CARD_DECLINED`,
  `PAYMENT_INSUFFICIENT_FUNDS`, and `PAYMENT_CALL_ISSUER` must go back to the
  buyer — never auto-retry them. `PAYMENT_TRANSIENT_ERROR` and
  `INVENTORY_RESERVATION_ERROR` may be retried with the same idempotency key.
  Full list: `errors/lemon-perfect-decline-codes.yml`.
- **Errors** are JSON-RPC 2.0 objects:
  `{ jsonrpc, id, error: { code, message, data: { code, content, continue_url } } }`.
  `data.continue_url` hands the buyer a human checkout to finish in a browser —
  use it rather than forcing the flow.

## Do not

- Do not handle raw card data. The store's payment handlers tokenize; you never
  touch a PAN.
- Do not treat the anonymous Storefront GraphQL cart mutations as a substitute
  for this flow to sidestep the approval gate.
- Do not assume order history is readable here — `get_order` covers orders in
  this flow; broader customer order history sits behind the OIDC-protected
  Customer Account API (`customer-account-api:full`).
