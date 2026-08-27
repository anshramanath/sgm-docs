# Architecture & Decisions (June 18, 2026)

Technical reference for the sunglass-monster server. Covers what was built, how it works, and why decisions were made.

---

## Stack

- **Next.js 16** — API routes only, no frontend pages. All routes live under `src/app/api/`.
- **Supabase** — Postgres database + auth token verification.
- **Stripe** — hosted checkout and webhooks for payment processing.

---

## Two-Client Pattern

Every request uses one of two Supabase clients:

| Client | Key | Use |
|--------|-----|-----|
| `createAdminClient()` | Service role | Public catalog reads — bypasses RLS |
| `createUserClient(req)` | Bearer token from `Authorization` header | Authenticated endpoints — RLS enforced |

`createUserClient` validates the token against Supabase and returns `{ supabase, user }`. Returns `null` if the token is missing or invalid — callers return 401.

Using the admin client for public endpoints means no token is required and no auth overhead. Using the user client for authenticated endpoints means RLS automatically scopes queries to the logged-in user — no manual `where user_id = x` needed.

---

## Response Shape

All endpoints use `ok()` and `err()` helpers from `src/lib/api.ts`:

```json
{ "success": true,  "data": { ... } }
{ "success": false, "error": "Message" }
```

---

## Database Design

### Brand scoping

Every table uses `brand_slug text references brands(slug) on delete cascade` rather than a `brand_id UUID`. This means:
- Queries filter directly on `brand_slug` — no join to brands needed.
- Deleting a brand cascades to all related rows automatically.

### Product structure

Products are either **simple** or **variable**:

- **Simple**: `products.sku` is set. No variations. Price and stock live at the product level.
- **Variable**: `products.sku` is null. Variations hold the SKUs, prices, stock, and images.

The same SKU string can appear across multiple products (e.g. two brands both use `SKU-BLK`). What uniquely identifies an item is `productName + sku` — or equivalently `productSlug + sku`. This is how the webhook and validate-cart both resolve line items.

### `total_sales`

- `variations.total_sales int not null` — always has a value.
- `products.total_sales int` — nullable, for simple products. `coalesce(total_sales, 0)` is used when incrementing.

### Bookmarks

Bookmarks store only `productSlug`, `name`, and `imageSrc` — no `sku`, no `attribute`. Reason: bookmarks can be added from list pages (homepage, category, sale) where products are shown at the product level. Variable products have no SKU at that level, so requiring one would make bookmarking impossible there.

There is no unique constraint on bookmarks. The frontend manages deduplication before calling PUT.

### Migration file conventions

| File | Role |
|------|------|
| `001_initial_schema.sql` | Source of truth for the full schema. Apply to a fresh DB to reproduce everything. Uses `create`. |
| `002_user_cart_bookmarks.sql` | Step record — historical snapshot of what was done at that point. Uses `create`. |
| `003_orders.sql` | Step record — orders, order_items, and both RPC functions. Uses `create`. |
| `00N_*.sql` (e.g. `004`, `005`) | Live-apply files — run once against the live DB, then delete. Use `create or replace` / `alter table`. |

---

## Stripe Webhook

Route: `POST /api/webhooks/stripe`

Handles `checkout.session.completed` only.

### Flow

1. Verify `stripe-signature` header.
2. Guard `brandSlug` from `session.metadata` (400 if missing).
3. **Idempotency check** — query `orders` by `stripe_session_id`. If row exists, return 200 immediately. Stripe retries on non-2xx, so returning 200 prevents duplicate processing.
4. Fetch line items from Stripe (expanded `price.product` for images).
5. Parse SKUs from line item descriptions. Format is `"Product Name (SKU)"` — SKU is extracted via `lastIndexOf("(")`.
6. **Parallel lookup** — query both `variations` and `products` (simple) to build a `skuMap` keyed by `"${productName}:${sku}"`. Each entry carries the row `id`, `productSlug`, and whether it's a `variation` or `simple` type.
7. Insert the `orders` row.
8. Loop line items → build `orderItems` array (skip any SKU that wasn't found in the catalog).
9. Insert `order_items`. **If insert fails → delete the order → return 500.** This gives Stripe a clean retry — if we left a committed order with no items, the idempotency check would find it on retry and return 200 without reinserting the items.
10. Loop `orderItems` → accumulate `Map<id, qty>` separately for products and variations.
11. Apply increments in parallel via Postgres RPCs.

### Why insert order_items before incrementing total_sales

`order_items` has a unique constraint on `(order_id, sku)`. This is the true idempotency gate — if the webhook fires twice and somehow gets past the `stripe_session_id` check, the second insert will fail and we catch it cleanly. Only after a successful insert do we touch `total_sales`.

### Atomic total_sales increment

Two Postgres RPCs handle incrementing:

```sql
create function increment_variation_total_sales(p_variation_id uuid, p_qty int)
returns void language sql as $$
  update variations set total_sales = total_sales + p_qty where id = p_variation_id;
$$;

create function increment_product_total_sales(p_product_id uuid, p_qty int)
returns void language sql as $$
  update products set total_sales = coalesce(total_sales, 0) + p_qty where id = p_product_id;
$$;
```

Why an RPC instead of read-modify-write in app code: if two webhook deliveries run concurrently and both read `total_sales = 3`, one writes 4 and the other writes 5 — the final value is 5 instead of 6. The RPC executes `SET total_sales = total_sales + qty` as a single atomic statement — Postgres serializes it correctly regardless of concurrency.

---

## Validate Cart

Route: `POST /api/public/validate-cart`

Checks whether each cart item still exists in the catalog before checkout. Each item requires `{ sku, productSlug }`.

- Queries both `variations` (joined to `products` for `product_slug`) and `products` (simple products) in parallel, filtered by all provided SKUs and product slugs.
- Builds a `Set` of `"productSlug:sku"` pairs found.
- Returns `{ sku, productSlug, exists }` for each input item.

`productSlug` is required (not just `sku`) because the same SKU string can exist in multiple products. Keying on both makes the check unambiguous.

---

## Order History

Route: `GET /api/user/orders?brandSlug=`

- Uses the user client — RLS on `orders` scopes to `auth.uid() = user_id` automatically.
- Single query with nested select: `orders` + `order_items` joined inline.
- Ordered newest first.

RLS on `order_items` uses an `exists` subquery against `orders` — no direct `user_id` column on items, access is inherited from the parent order.

---

## Checkout

Route: `POST /api/user/checkout`

Creates a Stripe checkout session. Idempotent — same cart contents and order count produce the same session URL (Stripe's idempotency key).

Idempotency key: `"${userId}:${hashCart(items)}:${orderCount}"`. The `orderCount` component ensures a second identical purchase after the first completes produces a new session rather than reusing the old one.

The success URL is passed in by the frontend. Stripe redirects there on payment completion. The webhook fires near-simultaneously and handles recording the order.

---

## Project Structure

```
src/
├── app/api/
│   ├── public/
│   │   ├── brands/route.ts
│   │   ├── categories/route.ts
│   │   ├── products/route.ts        # paginated by category, optional price/sale filter
│   │   ├── sale/route.ts            # always sale=true, optional price filter
│   │   ├── item/route.ts            # full product detail with variations
│   │   ├── search/route.ts          # ilike name search, limit 6
│   │   └── validate-cart/route.ts   # existence check before checkout
│   ├── user/
│   │   ├── cart/route.ts            # GET + PUT (full replace)
│   │   ├── bookmarks/route.ts       # GET + PUT (full replace)
│   │   ├── orders/route.ts          # GET order history
│   │   └── checkout/route.ts        # POST → Stripe session URL
│   └── webhooks/
│       └── stripe/route.ts          # checkout.session.completed handler
└── lib/
    ├── api.ts                        # ok() / err() helpers
    ├── stripe.ts                     # Stripe client
    ├── supabase/
    │   ├── admin.ts                  # service role client
    │   └── user.ts                   # user client from Bearer token
    └── db/
        ├── 001_initial_schema.sql    # full schema — source of truth
        ├── 002_user_cart_bookmarks.sql
        ├── 003_orders.sql
        └── reset_total_sales.sql     # testing utility
```
