---
name: Buy Hero Bread products as an agent
description: >-
  Search the Hero Bread catalog, build a cart, create and fulfill a checkout, and complete a
  purchase against the store's UCP/MCP endpoint — with the buyer approving payment.
api: mcp/hero-bread-mcp.yml
endpoint: https://shop.hero.co/api/ucp/mcp
operations:
  - search_catalog
  - get_product
  - create_cart
  - update_cart
  - create_checkout
  - update_checkout
  - complete_checkout
  - cancel_checkout
  - get_order
generated: '2026-08-22'
method: generated
source: mcp/hero-bread-ucp-mcp-tools-list.json + https://shop.hero.co/agents.md
---

# Buying from Hero Bread

Hero Bread's store speaks the Universal Commerce Protocol over MCP. Every tool name below was read
from a live `tools/list` against `https://shop.hero.co/api/ucp/mcp` — none are invented.

## Connect

POST JSON-RPC 2.0 to `https://shop.hero.co/api/ucp/mcp` with
`Content-Type: application/json` and `Accept: application/json, text/event-stream`.
`initialize` reports protocol version `2025-06-18`, server `universal-commerce`.

`initialize` and `tools/list` are anonymous. **Every `tools/call` is not.**

## Two things every call needs

1. `meta.ucp-agent.profile` — a resolvable https URI describing your agent. Omit it and the server
   answers `-32001 UCP discovery failed / invalid_profile_url`. Supply one the server cannot fetch
   and it answers `profile_unreachable`.
2. A Shopify agent JWT. Without it: `-32000 AuthenticationRequired`, pointing at
   `https://shopify.dev/docs/agents/get-started/authentication`.

## The flow

1. `search_catalog` — natural-language query and/or filters; at least one is required. Results are
   cursor-paginated; follow `pagination.cursor` for more.
2. `get_product` — full detail for one product: variants, exact price, real-time availability.
   Use `lookup_catalog` instead when resolving several `gid://shopify/Product/...` ids at once.
3. `create_cart`, then `update_cart` to adjust lines. `get_cart` reads it back.
4. `create_checkout` from the cart. `update_checkout` sets the shipping address and delivery method.
   This store ships to a **single destination only** (`allows_multi_destination.shipping: false`)
   and supports the `[shipping]` method combination.
5. `complete_checkout` — **requires `meta.idempotency-key`**. It is the only tool that does. Generate
   one per completion attempt and reuse it on retry; never on a new purchase.

## Money

Every price is an integer in ISO 4217 **minor units** with a currency code:
`{"amount": 2500, "currency": "USD"}` is $25.00. Divide by 100 for two-decimal currencies before
quoting a buyer. Zero-decimal currencies (JPY) are already whole units. Pass
`context.address_country` and `context.currency` so pricing and availability are right.

## The rule that is not optional

The store publishes it in both `llms.txt` and `robots.txt`: **checkout requires human approval.**
Do not complete payment without explicit, contemporaneous buyer consent — no scripted form fills, no
browser automation that finalizes a payment. If you cannot get approval at the moment of payment,
route the purchase through Shop Pay via `https://shop.app/SKILL.md` instead.

## Backing out

`cancel_cart` reverses a cart. `cancel_checkout` reverses a checkout **before** it is completed.
Neither has a published time window, so do not promise one to a buyer.

**After `complete_checkout` there is no reversal tool.** `get_order` is read-only and there is no
refund, void or cancel-order tool in the surface. A completed purchase is undone by a human under
the store's refund policy: https://shop.hero.co/policies/refund-policy — tell the buyer that before
they approve payment, not after.

## Rate limits

The endpoint is rate-limited per IP. No number, window or header is published. Back off on 429.
