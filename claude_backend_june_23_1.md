# Sunglass Server — What Was Built and Learned (June 23, 2026)

## What Was Built

A multi-brand e-commerce API server built with **Next.js App Router** (route handlers), **Supabase** (Postgres + auth), and **Stripe** (checkout + webhooks). The server supports multiple brands on a single deployment — every query is scoped by `brandSlug`.

---

## Endpoint Map

| Method | Path | Auth | Purpose |
|--------|------|------|---------|
| GET | `/api/public/brands` | No | List all brands |
| GET | `/api/public/categories` | No | Category tree for a brand |
| GET | `/api/public/products` | No | Paginated products by category |
| GET | `/api/public/sale` | No | Paginated sale products |
| GET | `/api/public/item` | No | Full product detail |
| GET | `/api/public/search` | No | Name search (up to 6 results) |
| POST | `/api/public/validate-cart` | No | Validate cart items + get current prices |
| GET | `/api/user/cart` | Yes | Fetch user's cart |
| PUT | `/api/user/cart` | Yes | Sync user's cart (replace) |
| GET | `/api/user/bookmarks` | Yes | Fetch user's bookmarks |
| PUT | `/api/user/bookmarks` | Yes | Sync user's bookmarks (replace) |
| POST | `/api/user/checkout` | Yes | Create Stripe checkout session |
| GET | `/api/user/orders` | Yes | User's order history |
| POST | `/api/webhooks/stripe` | Stripe sig | Handle `checkout.session.completed` |

---

## Database Schema

### Core catalog
```
brands            (id, name, slug)
categories        (id, brand_slug→brands, parent_id→categories, name, slug, sort_order)
products          (id, brand_slug→brands, name, slug, sku?, description, summary[], attributes jsonb,
                   featured, sale, min_price_cents, max_price_cents, sale_price_cents, total_sales)
product_categories (product_id→products, category_id→categories)  -- many-to-many
variations        (id, product_id→products, sku, attribute jsonb, sale, regular_price_cents,
                   sale_price_cents, total_sales)
product_images    (id, product_id→products, src, name, sort_order)
variation_images  (id, variation_id→variations, src, name, sort_order)
description_images (id, brand_slug→brands, src, name)
product_description_images (product_id→products, image_id→description_images)
```

### Orders
```
orders      (id, user_id→auth.users, brand_slug→brands, stripe_session_id unique,
             stripe_payment_intent unique, status, total_cents, shipping_address jsonb, created_at)
order_items (id, order_id→orders, product_slug, sku, name, image_src, price_cents, quantity,
             attribute jsonb)
```

### User data
```
cart_items  (id, user_id→auth.users, brand_slug→brands, product_id→products,
             product_slug, sku, attribute jsonb, name, image_src, price_cents, quantity, updated_at)
bookmarks   (id, user_id→auth.users, brand_slug→brands, product_id→products,
             product_slug, name, image_src, created_at)
admins      (user_id→auth.users)
```

### Notes
- All FK chains use `on delete cascade` so deleting a brand removes everything under it.
- `product_id` on `cart_items` and `bookmarks` is a hard FK to `products` — allows cascade delete when a product is removed, and is the foundation for a future slug-sync trigger.
- `products.sku` is nullable — only simple (non-variation) products have a SKU at the product level.
- Prices are always stored in cents (integer). Never floats.
- `attributes` and `attribute` are `jsonb` — structured but schemaless, validated in application code.

---

## Supabase: Two Clients

The most important architectural concept.

### Admin client (`createAdminClient`)
```typescript
createClient(url, SUPABASE_SERVICE_ROLE_KEY, {
  auth: { autoRefreshToken: false, persistSession: false }
})
```
- Uses the **service role key** — bypasses Row Level Security entirely.
- Use for: public endpoints, the products query in checkout, the webhook handler.
- Never expose this key to the frontend.

### User client (`createUserClient`)
```typescript
createClient(url, NEXT_PUBLIC_SUPABASE_ANON_KEY, {
  global: { headers: { Authorization: `Bearer ${token}` } }
})
```
- Uses the anon key + the user's JWT from `Authorization: Bearer <token>`.
- Supabase applies RLS policies, so the user can only see their own rows.
- `getUser()` validates the token and returns the authenticated user object.
- Returns `null` if the token is missing or invalid — use that as a 401 guard.

### When to use which in the same route
Checkout uses both: `adminSupabase` for the products query (bypasses RLS, reads any product), then `userSupabase` for the orders count (scoped to the authenticated user). The user client is created lazily, after the admin query and validation, so unauthenticated requests fail fast without a DB round-trip.

---

## Supabase Query Builder Patterns

### Nested select
```typescript
supabase.from("products").select(`
  id, name,
  variations(sku, attribute),
  product_images!inner(src, sort_order)
`)
```
- Nested relations use the table name in parentheses.
- `!inner` makes it a `JOIN` instead of a `LEFT JOIN` — rows with no matching images are excluded.

### Filtering on nested tables
```typescript
.eq("products.brand_slug", brandSlug)   // filter nested relation
.eq("brand_slug", brandSlug)            // filter root table
```

### Count without fetching rows
```typescript
.select("*", { count: "exact", head: true })
```
- `head: true` sends a HEAD request — returns `count` but no rows.
- Used in checkout to get `orderCount` for the idempotency key.

### `.single()` vs array
- `.single()` returns `data` as an object and errors if 0 or 2+ rows match.
- Default returns `data` as an array.

---

## Price Enforcement

Frontend-sent prices are never trusted. The checkout route ignores any `priceCents` from the request body. Instead:

1. Query DB for all matching products and variations using `productSlug` values from the cart.
2. Build a composite-key price map:
   ```typescript
   const priceMap = new Map<string, number>();
   // Key: "productSlug:sku" → Value: price in cents
   for (const p of productRows) {
     if (p.sku) priceMap.set(`${p.slug}:${p.sku}`, p.sale ? p.sale_price_cents : p.min_price_cents);
     for (const v of p.variations) priceMap.set(`${p.slug}:${v.sku}`, v.sale ? v.sale_price_cents : v.regular_price_cents);
   }
   ```
3. The `sale` boolean on the row determines which price to use — not a calculation.
4. `unit_amount` passed to Stripe is always `priceMap.get(key)`, never from the request.

The same priceMap logic is shared between `validate-cart` and `checkout`.

---

## Composite Key Pattern

Using `"slug:sku"` as a Map key gives O(1) lookup over a flat structure instead of nested Maps or repeated `.find()` calls.

```typescript
const key = `${item.productSlug}:${item.sku}`;
const exists = priceMap.has(key);
const price = priceMap.get(key) ?? null;
```

Flat composite key is simpler than `Map<string, Map<string, number>>` which causes type ambiguity and harder lookups.

---

## Validate-Cart and 409

`POST /api/public/validate-cart` validates every item and returns current DB prices in a single query. Status code communicates the aggregate result:

- `200` — all items exist
- `409 Conflict` — one or more items no longer exist in the catalog

This 409 is intentional and readable in logs — it signals stale cart data, not an error. The frontend has to check prices regardless, but the status code lets server logs distinguish clean checkouts from stale ones without parsing response bodies.

---

## Stripe Checkout

### Creating a session
```typescript
stripe.checkout.sessions.create({
  mode: "payment",
  client_reference_id: user.id,   // stored, used in webhook to link order to user
  customer_email: user.email,
  metadata: { brandSlug },        // stored on session, accessible in webhook
  line_items: items.map(item => ({
    price_data: {
      currency: "usd",
      product_data: {
        name: `${item.name} — ${item.attribute.map(a => a.option).join(" / ")}`,
        description: item.sku,   // SKU stored here for webhook parsing
        images: [item.imageSrc],
      },
      unit_amount: priceMap.get(`${item.productSlug}:${item.sku}`),
    },
    quantity: item.quantity,
  })),
  shipping_address_collection: { allowed_countries: ["US"] },
  success_url: successUrl,
  cancel_url: cancelUrl,
}, { idempotencyKey })
```

### Idempotency key
```typescript
const idempotencyKey = `${user.id}:${hashCart(items)}:${orderCount}`;
```
- Prevents duplicate sessions if the same request is retried.
- Includes `orderCount` so a second checkout after a completed order gets a fresh session.
- Cart is hashed (SHA-256, first 32 chars) — same items in any order produce the same key.

### Stripe passes cents
Stripe's `unit_amount` is in the smallest currency unit — cents for USD. So 1650 = $16.50. Never divide or multiply; store and pass cents directly.

---

## Webhook Handler

Handles `checkout.session.completed` only. All other event types return 200 immediately.

Flow:
1. Verify `stripe-signature` header with `stripe.webhooks.constructEvent`.
2. Check for existing order by `stripe_session_id` — if found, return 200 (idempotent).
3. Expand line items with `expand: ["data.price.product"]` to access product images.
4. Parse SKU from `item.price.product.description` (where checkout stored it).
5. Query DB for variations and simple products by SKU to build a lookup map.
6. Insert `orders` row using `session.client_reference_id` (user ID) and `session.metadata.brandSlug`.
7. Insert `order_items` rows from line items.
8. Atomically increment `total_sales` on relevant products/variations via RPC.
9. If `order_items` insert fails, delete the orphaned `orders` row before returning 500.

### Idempotency
The `stripe_session_id` unique constraint ensures duplicate webhook deliveries don't create duplicate orders. The early-exit check on step 2 handles this.

---

## Cart and Bookmarks: Sync Pattern

Both use a **replace** strategy — no partial updates.

```
PUT /api/user/cart   { brandSlug, items: [...] }
```

1. Delete all existing rows for `user_id + brand_slug`.
2. Insert the new set.

This keeps the frontend as the source of truth for cart/bookmark state. The server just persists whatever the frontend sends. No merge logic, no conflict resolution.

---

## Category Tree

Categories are stored flat with a `parent_id` self-reference. The tree is built in application code:

1. Fetch all categories for the brand.
2. Build a `nodeMap` by `id`.
3. Walk rows: if `parent_id` exists, attach to parent's `children`; otherwise push to roots.
4. Recursively sort each level by `sortOrder`.

This avoids recursive SQL CTEs and is fast enough for typical category counts.

---

## Response Shape

All endpoints use the same envelope via `ok()` and `err()`:

```json
{ "success": true,  "data": ... }
{ "success": false, "error": "Message" }
```

`data` is always a flat value — never nested inside another key like `{ data: { items: [...] } }`. Arrays are returned directly, not wrapped in an object, except for paginated endpoints which need metadata alongside the array.

---

## Pagination

Used on `/products` and `/sale`. Query params: `page` (default 1), `size` (default 20, max 100).

```typescript
const from = (page - 1) * size;
const to = from + size - 1;
// ...
.range(from, to)
```

Response always includes `{ products, page, size, totalPages, totalProducts, hasNextPage }`.

---

## Color Deduplication in Listings

Products can have many variations (e.g., black/standard, black/large, blue/standard, blue/large). The listing pages only want one swatch per color.

```typescript
const seen = new Set<string>();
const variations = rawVariations.reduce((acc, v) => {
  const color = v.attribute.find(a => a.name === "color");
  if (!color || seen.has(color.slug)) return acc;
  seen.add(color.slug);
  acc.push({ id: v.id, option: color.option, slug: color.slug, value: color.value, ... });
  return acc;
}, []);
```

The variation `id` is included so the frontend can use it as a React list key. First image from the first variation of that color is used as the swatch image.

---

## Variation Attributes: Two Levels of Detail

**In listings** (`/products`, `/sale`): each variation is one color swatch — `{ id, option, slug, value, imageSrc, imageName }`.

**In item detail** (`/item`): each variation carries its full attribute array, but simplified to `{ name, slug }` only:
```typescript
attribute: (v.attribute as RawAttr[]).map(a => ({ name: a.name, slug: a.slug }))
```
Display labels and hex values live in the top-level `attributes` array on the product. The frontend looks up display info by slug — no duplication across every variation.

---

## RLS (Row Level Security)

User-owned tables (`orders`, `order_items`, `cart_items`, `bookmarks`) have RLS enabled. Policies use `auth.uid()`:

```sql
create policy "cart_items: users manage own rows"
  on cart_items for all
  using (auth.uid() = user_id)
  with check (auth.uid() = user_id);
```

The user Supabase client (anon key + JWT) respects these policies automatically. The admin client (service role) bypasses them entirely — which is why the webhook and public product queries use it.

---

## Known Pending Issues

1. **Webhook SKU parsing** — currently uses `lastIndexOf("(")` on `item.description` (old format). Should read `item.price.product.description` directly since that's where the SKU is now stored.
2. **Cart/bookmark `product_id`** — schema has `product_id NOT NULL` FK but the PUT sync endpoints don't include it in the insert rows. Frontend needs to send `productId` and backend needs to pass it through.
3. **Slug sync trigger** — `product_id` FK was added to support cascade deletes and a future trigger to keep `product_slug` in sync when a product's slug changes. Trigger not yet written (deferred until admin endpoints exist).

