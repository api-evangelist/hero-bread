---
name: Read the Hero Bread catalog without transacting
description: >-
  Discover Hero Bread products, prices and availability read-only — via anonymous MCP discovery or
  the storefront JSON paths the store documents for crawling agents.
api: mcp/hero-bread-mcp.yml
endpoint: https://shop.hero.co/api/ucp/mcp
operations:
  - search_catalog
  - lookup_catalog
  - get_product
generated: '2026-08-22'
method: generated
source: mcp/hero-bread-ucp-mcp-tools-list.json + https://shop.hero.co/llms.txt + https://shop.hero.co/robots.txt
---

# Reading the Hero Bread catalog

Two paths. Pick by whether you can present an agent identity.

## Path A — MCP (needs identity)

`POST https://shop.hero.co/api/ucp/mcp`

- `tools/list` answers **anonymously** — use it to confirm the surface is up and see the schemas.
- `search_catalog`, `lookup_catalog` and `get_product` are `tools/call`, so they need
  `meta.ucp-agent.profile` and a Shopify agent JWT like every other tool.

`search_catalog` takes a natural-language `query`, structured `filters`, or both — at least one is
required. Paginate with `pagination.cursor`.

## Path B — storefront HTTP (no identity, no auth)

The store's own `llms.txt` publishes these for read-only agents:

- `GET https://shop.hero.co/collections/all` — browse everything
- `GET https://shop.hero.co/products/{handle}` — product page
- `GET https://shop.hero.co/products/{handle}.json` — product as JSON
- `GET https://shop.hero.co/collections/{handle}/products.json` — collection as JSON
- `GET https://shop.hero.co/search?q={query}&type=product`
- `GET https://shop.hero.co/sitemap.xml`

Verified live on 2026-08-22: `/products.json?limit=3` returns HTTP 200,
`application/json`.

## What robots.txt forbids

Do not use `/cart.js` or `/recommendations/products` — both are `Disallow` and the store says
explicitly that agents should use UCP/MCP for cart and checkout instead. `/checkout`, `/checkouts/`,
`/orders`, `/account` and `/admin` are all `Disallow`.

## Prices

Storefront JSON returns prices as decimal strings; the MCP surface returns integer minor units with
a currency code. They are the same money in two shapes — do not mix them in one calculation.

## Store policies

The legacy Shopify Storefront MCP server at `https://shop.hero.co/api/mcp` exposes a
`search_shop_policies_and_faqs` tool that the newer UCP surface does not. If you need policy or FAQ
answers as a tool rather than a page, that is where it lives. Otherwise read
`https://shop.hero.co/policies/refund-policy`, `/policies/privacy-policy`,
`/policies/terms-of-service`, or `https://www.hero.co/faq`.
