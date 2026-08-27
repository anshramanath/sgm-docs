# Learnings & Architecture (June 23, 2026)

Everything built, every decision made, and why.

---

## Project Overview

Multi-brand sunglass e-commerce platform. A single Next.js server serves multiple brands — all endpoints are scoped by `brandSlug`. Supabase for the database and auth, Stripe for payments.

---

## Database Schema (001_initial_schema.sql is source of truth)

### Catalog
- `brands` — top level, everything hangs off `brand_slug`
- `categories` — tree structure via `parent_id`, sorted by `sort_order`
- `products` — has optional `sku` (only set for simple products with no variations), `total_sales int`
- `variations` — each has a `sku`, `attribute jsonb` (array of `{ name, option, slug, value? }`), `total_sales int not null`
- `product_images`, `variation_images` — sorted by `sort_order`, products always have at least one
- `description_images` + `product_description_images` — join table for rich description images

### User
- `cart_items` — `product_id` FK for cascade delete, stores full denormalized item data (name, imageSrc, priceCents, attribute)
- `bookmarks` — same `product_id` FK pattern, no price/quantity
- `orders` — `stripe_session_id unique` for idempotency, `status text not null` (no default — always set explicitly)
- `order_items` — `attribute text` (nullable — display string for variations, null for simple products)
- `admins` — simple table, `user_id` only

### Key Schema Decisions
- `order_items.attribute` is `text` not `jsonb` — it's a display string derived from the Stripe product name at webhook time, not structured data
- `cart_items.attribute` is `jsonb not null` — structured `[{ name, option }]` array needed for cart display
- No defaults on `quantity`, `attribute`, or `status` — caller always provides these explicitly
- `product_id` FK on cart_items and bookmarks enables cascade delete when a product is removed
- SKUs are unique within a brand (confirmed via SQL) — safe to use `productSlug:sku` as a composite key

---

## Supabase Clients

Two clients with distinct roles:

| Client | Key | Bypasses RLS | Used For |
|--------|-----|--------------|---------|
| `createAdminClient()` | Service role | Yes | Public endpoints, webhook inserts |
| `createUserClient(req)` | Anon + JWT | No | Authenticated user endpoints |

`createUserClient` extracts the JWT from `Authorization: Bearer <token>`, returns `{ supabase, user }` or null if invalid.

---

## Public Endpoints

### Products / Catalog
- **brands** — flat list
- **categories** — tree built in JS from flat DB rows using a node map, sorted recursively
- **products** — paginated by category, color dedup in JS (one swatch per unique color slug)
- **sale** — same as products filtered to `sale = true`, no `sale` field in response (implied)
- **item** — full product detail: all variations, all images, description images, structured attributes
- **search** — `ilike` name search, limit 6, no pagination

### Product Images
Products are guaranteed to have at least one image — `firstImage!.src` (non-null assertion, no fallback). Variation images are optional — `firstImage?.src ?? null`.

### validate-cart
Checks existence and price freshness for cart items before display and before checkout.

**4 status codes:**
| Status | Meaning |
|--------|---------|
| `200` | All good |
| `404` | One or more items don't exist |
| `409` | One or more prices changed |
| `422` | Both missing items and changed prices |

Input `priceCents` is compared against the current DB price. `priceChanged: true` signals the frontend to show an updated price, not just "item missing."

### Price Map (validate-cart)
Built with composite key `productSlug:sku` → `priceCents`. O(1) lookup per cart item. Only fetches products in the cart (`.in("slug", slugs)`). Map contains all variations for each fetched product — slightly more keys than needed but one round trip.

---

## Authenticated Endpoints

### Cart
- GET returns flat array with `productId`
- PUT is full replace (delete + insert), not atomic but the failure window is milliseconds
- Stores denormalized data: name, imageSrc, priceCents, attribute — so cart renders without additional queries
- `product_id` FK means if a product is deleted, cart rows are automatically removed

### Bookmarks
- Same pattern as cart but no price/quantity/sku
- `productId` FK for same cascade behavior

### Orders
- GET returns flat array, newest first, scoped by `brand_slug` + RLS user scoping
- `attribute` passed through as-is: display string or null
- No `sku` in response — not useful to the customer

### Checkout
**What the frontend sends:** `{ productSlug, sku, priceCents, quantity }` only — no name, image, or attribute.

**What the server does:**
1. Fetches products + variations from DB (admin client, bypasses RLS)
2. Builds `entryMap: Map<"slug:sku", Entry>` where `Entry = { priceCents, name, attribute, imageSrc }`
3. Validates: checks existence and price match — returns 404/409/422 if invalid
4. Auth check happens AFTER validation so unauthenticated users get the same price-change errors
5. Hashes the cart using DB values (not frontend values) for idempotency key
6. Creates Stripe session using DB prices — frontend price is never trusted for charging
7. Returns URL string directly (not wrapped)

**Idempotency key:** `userId:hash(DB cart state):orderCount`
- Hash includes DB priceCents, name, attribute, imageSrc — any DB change produces a new session
- `orderCount` ensures post-order checkout of same items gets a fresh session
- Prevents a stale session with old prices from being reused

**Price hack protection:** If a price changes after a session URL is created, the next checkout call produces a new session. A user can't pay the old price by reusing a stale URL because the idempotency key changes.

### Entry Map vs Price Map
- `validate-cart` uses a `priceMap` (slug:sku → priceCents only)
- `checkout` uses an `entryMap` (slug:sku → full Entry object) — needed for name, attribute, imageSrc in Stripe line items and the hash

---

## Webhook

Handles `checkout.session.completed`. No DB query for product data — everything read from Stripe's expanded line items.

**Flow:**
1. Verify signature
2. Check `stripe_session_id` uniqueness — idempotent, duplicate deliveries ignored
3. Insert `orders` row with `status: "processing"`
4. Derive `product_slug` via `slugify(name)` from Stripe product name
5. Derive `attribute` by splitting product name on ` — `: `"Sport Sunglasses — Gloss Black / Standard"` → `"Gloss Black / Standard"`, no ` — ` → `null`
6. Insert `order_items`
7. Trigger fires automatically → increments `total_sales`

**Stripe product name format:** `"${name} — ${options}"` where options = `attribute.map(a => a.option).join(" / ")`. Simple products: just `name`.

**SKU location:** `product.description` (not the name)

**Shipping address:** Always present — we enforce `shipping_address_collection` in checkout, so `!` assertions are safe in the webhook.

**On items insert failure:** Deletes the order row manually (not atomic — if delete also fails, order exists with no items, an edge case the merchant handles).

---

## Total Sales Trigger

A Postgres trigger on `order_items` AFTER INSERT automatically increments `total_sales`.

```sql
create trigger order_items_update_total_sales
  after insert on order_items
  for each row execute function update_total_sales();
```

**Logic:**
1. Get `brand_slug` from the parent order (needed to scope product lookup — `product_slug` alone isn't brand-unique)
2. Inner join `variations` + `products` filtering on `product_slug` + `brand_slug` + `sku` → get `variation_id` if match
3. If match → increment `variations.total_sales`
4. If no match → increment `products.total_sales` where `slug + brand_slug + sku` matches (simple product)
5. If neither matches (product deleted) → silent no-op

**Why a trigger:** Webhook just inserts rows — no RPC calls needed. Increment is guaranteed on every insert regardless of where it comes from. Two targeted SQL queries (join then fallback) vs one query + JS map — right tool for the context.

**Deleted product edge case:** If a product is deleted after a Stripe session was created but before the user pays, the webhook still creates the order (Stripe already charged), the trigger silently skips, and `total_sales` isn't incremented. Merchant sees an order for a nonexistent product and processes a refund. Correct behavior — can't block a completed payment.

---

## Key Patterns & Decisions

### Composite key `productSlug:sku`
Used as Map keys throughout. Safe because SKUs are unique within a brand. Enables O(1) lookup without a second DB query.

### `??` vs `?.` vs `!`
- `?.` — optional chaining, returns `undefined` if key doesn't exist
- `??` — nullish coalescing, fallback for `null` or `undefined`
- `!` — non-null assertion, used when data is guaranteed (product images, Stripe shipping address)

### Supabase `.eq()` on joined tables
`.eq("related_table.column", value)` is an existence filter on the parent row — "include this parent if at least one child satisfies the condition." It does NOT filter which child rows are returned. To filter child rows, filter in JS after the query.

### Single query vs two queries
In JS endpoints, one query fetches all relevant variations and builds a Map — one round trip, slightly more data. In the SQL trigger, two targeted queries (join then fallback) — one or two round trips but no unnecessary data. Right tool for each context.

### Auth after validation in checkout
Unauthenticated users hit the products query and validation first. This means they get the correct 404/409/422 error even before auth is checked — better UX than hitting a 401 when the cart is actually stale.

### `order_items.attribute` as text
Stored as a plain display string (`"Gloss Black / Standard"`) derived from splitting the Stripe product name. `null` for simple products. Avoids re-parsing structured data for what is purely a display field in order history.

### Status lifecycle
`processing` → `shipped` → `delivered`. Set explicitly in the webhook — no DB default. Admin endpoints (not yet built) will handle status updates.

---

## Migration Files
| File | Purpose |
|------|---------|
| `001_initial_schema.sql` | Source of truth — full current schema |
| `002_user_cart_bookmarks.sql` | Historical — when cart/bookmarks were added |
| `003_orders.sql` | Historical — when orders/order_items were added |
| `drop_schema.sql` | Dev only — wipe and re-apply 001 |
