---
name: shopify
description: |
  Headless Shopify Plus, Storefront API, checkout, subscriptions, webhooks, and product catalog management. Use when building or reviewing Shopify commerce integrations.
  Triggers: "Shopify", "Storefront API", "headless commerce", "Shopify checkout", "Shopify webhook", "product catalog", "Shopify Plus"
lastReviewed: 2026-04-10
upstreamDeps: [shopify]
---

# Shopify Integration

Use this skill when building or reviewing headless Shopify integrations — Storefront API, checkout flows, cart management, subscriptions, webhooks, and product catalog patterns. These patterns apply regardless of the frontend framework.

## Core principles

1. **Storefront API for the frontend, Admin API for the backend** — The Storefront API is designed for public-facing storefronts (products, carts, checkout). The Admin API is for server-side operations (order management, inventory, customer data). Never expose Admin API tokens to the client.
2. **Cart state is sacred** — Cart data must persist across sessions, devices, and authentication states. A lost cart is a lost sale.
3. **Webhook-driven sync** — Product data, inventory, and order status should sync via webhooks. Never poll the API for changes.
4. **Checkout is Shopify's domain** — Use Shopify's hosted checkout or Checkout API. Never handle raw payment card data.

## Storefront API

### Authentication
- Use a public Storefront API access token (safe for client-side use)
- Use a private access token for server-side requests (higher rate limits, more fields)
- Scope tokens to the minimum required permissions

### Product queries
- Fetch products with variants, images, pricing, and availability in a single query
- Use `availableForSale` to filter products that are in stock
- Handle multi-currency with `presentmentPrices` or the `@inContext` directive
- Use pagination with cursors for large catalogs (`first`, `after`)

### Cart management
- Create carts with `cartCreate` mutation
- Store the cart ID in an `httpOnly` cookie for persistence
- Use `cartLinesAdd`, `cartLinesUpdate`, `cartLinesRemove` for modifications
- Apply discount codes with `cartDiscountCodesUpdate`
- Always check `cart.checkoutUrl` for the latest checkout link

### Checkout
- Redirect to Shopify's hosted checkout via `cart.checkoutUrl`
- For custom checkout (Shopify Plus), use the Checkout API
- Support discount codes, gift cards, and shipping rate selection
- Implement abandoned cart recovery via Shopify's built-in tools or webhooks

### Metafields
- Use metafields for custom product data (sizing guides, care instructions, specs)
- Query metafields in the same request as the product
- Define metafield types in Shopify admin for validation

## Webhooks

### Setup
- Register webhooks via the Admin API or Shopify CLI
- Always verify webhook HMAC signatures before processing
- Implement idempotent handlers (webhooks can be delivered more than once)
- Respond with `200` quickly — process heavy work asynchronously

### Key events
- `products/update` — trigger revalidation of product pages
- `inventory_levels/update` — update stock status in real-time
- `orders/create` — trigger fulfillment workflows
- `customers/create` — sync customer data to external systems
- `app/uninstalled` — clean up app data

### Revalidation pattern
- On `products/update` webhook, call on-demand revalidation for the affected product page
- On `collections/update`, revalidate collection pages
- Batch revalidation for bulk updates to avoid rate limits

## Subscriptions (Shopify Plus)

- Use the Selling Plan API for subscription products
- Display subscription options alongside one-time purchase
- Show the subscription discount and billing frequency clearly
- Handle subscription management (pause, skip, cancel) via the Customer Account API or a custom portal

## Multi-market and localization

- Use the `@inContext` directive for country and language-specific pricing
- Handle currency conversion and display with Shopify Markets
- Implement `hreflang` tags for multi-language storefronts
- Use Shopify's automatic translation or integrate with a translation service

## Review checklist

- Storefront API token is public; Admin API token is server-side only
- Cart persists across sessions via cookie-based cart ID
- Webhook HMAC signatures are verified
- Webhook handlers are idempotent
- Product pages include JSON-LD structured data
- Out-of-stock products handled gracefully
- Checkout uses Shopify's hosted checkout or Checkout API
- Product images are optimized and responsive
- Revalidation triggered by webhooks (not time-based polling)
- Multi-currency handled correctly if applicable

## Common issues to flag

- Admin API token exposed to the client
- Cart stored in `localStorage` instead of server-side cookies
- Missing webhook signature verification
- Non-idempotent webhook handlers (duplicate processing)
- Hardcoded prices or product data instead of API-fetched
- Missing structured data on product pages
- No revalidation strategy (stale product data after updates)
- Handling raw payment data instead of using Shopify checkout
- Ignoring `availableForSale` status (selling out-of-stock items)
