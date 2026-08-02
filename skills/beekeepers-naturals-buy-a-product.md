---
name: Buy a product from Beekeeper's Naturals with buyer approval
description: Search the Beekeeper's Naturals catalog, build a cart, run a checkout, and complete the purchase
  only after an explicit, contemporaneous buyer approval — using the store's UCP Shopping MCP endpoint.
api: mcp/beekeepers-naturals-mcp.yml
generated: '2026-08-02'
method: generated
source: https://www.beekeepersnaturals.com/agents.md
operations:
  - search_catalog
  - get_product
  - create_cart
  - create_checkout
  - update_checkout
  - complete_checkout
---

# Buy a product from Beekeeper's Naturals

Every tool name below exists in the UCP Shopping OpenRPC schema that this store's own
`/.well-known/ucp` merchant profile names as the schema for its MCP service. Nothing here is invented.

## Endpoint and identity

- **Discovery:** `GET https://www.beekeepersnaturals.com/.well-known/ucp` — confirm the version you intend
  to speak is in `supported_versions` (currently `2026-04-08` and `2026-01-23`) and read
  `services["dev.ucp.shopping"]` for the MCP `endpoint`.
- **MCP endpoint:** `POST https://www.beekeepersnaturals.com/api/ucp/mcp`,
  `Content-Type: application/json`, JSON-RPC 2.0.
- **Agent identity is mandatory.** Every call must carry a `meta["ucp-agent"]["profile"]` URI (HTTP
  `UCP-Agent` header) pointing at your platform's published UCP profile document. Without it the server
  returns `-32001 UCP discovery failed` / `invalid_profile_url` and nothing — including `tools/list` —
  will work.

## Steps

1. **Confirm capabilities.** `GET /.well-known/ucp`. Check that
   `capabilities["dev.ucp.shopping.checkout"]` and `["dev.ucp.shopping.cart"]` are present at your
   version, and note the available `payment_handlers` (`gpay`, `shopify.card`, `shop_pay`).
2. **Find the product.** Call `search_catalog` with the buyer's intent. Pass
   `context.address_country` and `context.currency` so prices and availability are correct for the buyer.
   If you already have a handle or identifier, call `get_product` or `lookup_catalog` instead.
   The same catalog is readable anonymously at `GET /products/{handle}.json` and
   `GET /collections/{handle}/products.json` if you only need to read.
3. **Build the cart.** Call `create_cart` with the chosen variant(s), then `get_cart` / `update_cart` to
   adjust quantities. Do **not** script `/cart.js` — `robots.txt` disallows it and directs agents to
   UCP/MCP.
4. **Open a checkout.** Call `create_checkout`, then `update_checkout` to set the shipping address and
   shipping method. Fulfillment on this store allows a single shipping destination
   (`allows_multi_destination.shipping: false`, `allows_method_combinations: [["shipping"]]`).
5. **Get explicit buyer approval.** This is a hard rule from the store: *"Checkout requires human approval.
   Agents must not complete payment without explicit buyer consent."* If you cannot obtain contemporaneous
   approval at the moment of payment, stop and route the purchase through Shop Pay via the Shop skill
   (`https://shop.app/SKILL.md`) instead of completing it yourself.
6. **Complete the purchase.** Call `complete_checkout`. `meta["idempotency-key"]` (a UUID) is **required**
   — generate one per logical purchase attempt and reuse it verbatim on any retry, so a network retry
   cannot double-charge the buyer. The same requirement applies to `cancel_checkout` and `cancel_cart`.
7. **Confirm.** Call `get_order` to read the placed order back and report the confirmation to the buyer.

## Rules to obey

- **Never finalize payment autonomously.** No scripted form fills, no browser automation of `/checkout`,
  no end-to-end flow that places an order without a human approving at the moment of payment.
- **Idempotency is not optional on the destructive calls.** `complete_checkout`, `cancel_checkout` and
  `cancel_cart` all require `meta["idempotency-key"]`.
- **Back off on 429.** The MCP endpoint is rate-limited per IP.
- **Errors are JSON-RPC error objects**, not RFC 9457 problem documents — branch on `error.data.code`.
  See `errors/beekeepers-naturals-problem-types.yml`.
- **Buyer account data** (order history, saved addresses) lives behind Shopify customer-account OAuth with
  the `customer-account-api:full` / `customer-account-mcp-api:full` scopes — see
  `scopes/beekeepers-naturals-scopes.yml`. That is a separate authorization from your UCP agent identity.
