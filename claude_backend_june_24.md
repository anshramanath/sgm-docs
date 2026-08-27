# LEARNINGS (June 24, 2026)

Everything built and learned on the sunglass-server backend.

---

## What Was Built

A Next.js API server for a multi-brand sunglass e-commerce platform. Deployed on Vercel, backed by Supabase (Postgres + Auth + Storage) and Stripe.

**Endpoints**

| Method | Path | Auth | Purpose |
|--------|------|------|---------|
| GET | `/api/public/brands` | no | list all brands |
| GET | `/api/public/categories` | no | full category tree for a brand |
| GET | `/api/public/products` | no | paginated products for a category |
| GET | `/api/public/sale` | no | paginated sale products |
| GET | `/api/public/item` | no | full product detail with variations/images |
| GET | `/api/public/search` | no | product name search (up to 6 results) |
| POST | `/api/public/validate-cart` | no | check items exist + prices match |
| POST | `/api/user/cart` | yes | get user's cart items |
| PUT | `/api/user/cart` | yes | replace cart (delete + insert) |
| POST | `/api/user/bookmarks` | yes | get user's bookmarks |
| PUT | `/api/user/bookmarks` | yes | replace bookmarks |
| POST | `/api/user/orders` | yes | get order history |
| POST | `/api/user/checkout` | yes | create Stripe checkout session |
| POST | `/api/webhooks/stripe` | stripe sig | handle checkout.session.completed |

---

## Architecture

### Two Supabase Clients

```ts
createAdminClient()   // service role key — bypasses RLS, used in public endpoints and webhook
createUserClient(req) // anon key + user's JWT — respects RLS, returns { supabase, user } or null
```

Public endpoints use the admin client to read catalog data without needing a user token.
User endpoints use the user client so RLS policies automatically scope every query to the logged-in user.
The webhook uses the admin client because Stripe has no user token.

### Response Shape

Every endpoint uses `ok()` and `err()` from `src/lib/api.ts`:

```ts
ok(data, status)  → { success: true,  data: <whatever> }
err(msg,  status) → { success: false, error: "Message" }
```

- Single resource → `data` is an object: `{ "url": "..." }`
- Multiple resources → `data` is an array: `[{ ... }, { ... }]`
- Paginated → `data` is an object with an array field + metadata: `{ products: [...], page, totalPages, ... }`
- Never a bare primitive in `data`

### Auth Flow

1. User logs in via Supabase Auth (email/password). Supabase sets a cookie with the JWT.
2. Frontend reads the JWT from the cookie and sends it as `Authorization: Bearer <token>`.
3. `createUserClient` extracts the token, initializes a Supabase client with it, calls `auth.getUser()` to verify it, and returns `{ supabase, user }`.
4. RLS policies on `cart_items`, `bookmarks`, `orders`, and `order_items` automatically filter to `auth.uid() = user_id`.
5. JWT is valid for 1 hour.

---

## Schema Decisions

### products

- `sku text` — nullable. Only set if the product has no variations (simple product). Variations carry their own SKUs.
- `total_sales int` — initialized to 0, incremented by the `update_total_sales` trigger.
- `min_price_cents` / `max_price_cents` — denormalized from variations for fast list queries.
- `attributes jsonb` — array of `{ name, options: [{ option, slug, value? }] }`. Defines what attributes (color, size, etc.) the product has, with display labels and color hex values. Separate from `variations`.

### variations

- `attribute jsonb` — array of `{ name, slug }` pairs. References slugs in `products.attributes`. No display labels — use the parent product's attributes to look them up.
- `total_sales int not null` — always initialized to 0.

### order_items

- `attribute text` — nullable. Display string for variation products (e.g. `"Gloss Black / Standard"`). `null` for simple products. Stored as a string because it's display-only; no need to parse it after the fact.
- No `default` on `status` in `orders` — the application always sets it explicitly to `"processing"`.

### cart_items

- `attribute jsonb not null` — array of `{ name, option, slug }`. Stores display labels (option) and URL slugs (slug) so the frontend can both display the selection and link to the product page without a lookup.
- No `default` or `check` on `quantity` — the application enforces valid values.

---

## Validate Cart + Checkout

### validate-cart

One DB query fetches all matching products (and their variations) by brand + slug list. A `Map<"slug:sku", priceCents>` is built for O(1) lookup per item. Returns per-item `{ exists, priceCents, priceChanged }`.

**Status code logic:**
| Condition | Status |
|-----------|--------|
| all good | 200 |
| missing items only | 404 |
| price changed only | 409 |
| both | 422 |

Checkout runs the same validation before creating a Stripe session.

### Composite key pattern

`"productSlug:sku"` is used as the Map key throughout validate-cart and checkout. This avoids nested Maps and gives O(1) lookup. It works because `(brand_slug, slug)` is unique on products and `sku` is unique within a product.

### Why the Map has extra entries

The query fetches all variations for each matching product, not just the ones in the cart. So the Map is larger than needed — but it's still one query, and the extra entries are just ignored. The alternative (filtering to exact slug+sku pairs in SQL) requires two queries in the worst case (first check variations, then fall back to product SKU).

### Checkout idempotency

Stripe checkout sessions are created with an `idempotencyKey = userId:hash(cart):orderCount`. The hash is a SHA-256 of the canonicalized cart (sorted by SKU, using DB prices). If the user opens checkout twice with the same cart and no new orders, the same Stripe session is returned. Any change to products, prices, or a new order breaks the key and creates a fresh session.

### Price enforcement

Prices in the checkout session come from the DB — the frontend `priceCents` is only used for comparison during validation. The Stripe line items always use `entry.priceCents` from the server's entryMap.

---

## Webhook

`POST /api/webhooks/stripe` handles `checkout.session.completed`.

1. Verify `stripe-signature` header with `stripe.webhooks.constructEvent`.
2. Check if `orders` row with this `stripe_session_id` already exists — if yes, return 200 (idempotent).
3. Fetch expanded line items from Stripe.
4. Insert `orders` row: status = `"processing"`, shipping address from `session.collected_information.shipping_details`.
5. Insert `order_items` rows — one per Stripe line item.
6. If order_items insert fails, delete the orders row and return 500 (keep DB consistent).

### Recovering product info from line items

At checkout session creation time, the product name is encoded as `"Name — Attribute"` and the SKU is stored in `description`. The webhook splits on `" — "` to recover name and attribute:

```ts
const [name, attribute] = product.name.split(" — ");
// attribute is undefined for simple products → stored as null
```

`slugify(name)` reconstructs `product_slug`.

---

## Total Sales Trigger

A Postgres `AFTER INSERT` trigger on `order_items` increments `total_sales` on the right row.

```sql
create or replace function update_total_sales()
returns trigger language plpgsql as $$
declare
  v_brand_slug text;
  v_variation_id uuid;
begin
  select brand_slug into v_brand_slug from orders where id = new.order_id;

  select v.id into v_variation_id
  from variations v
  join products p on p.id = v.product_id
  where p.slug = new.product_slug
    and p.brand_slug = v_brand_slug
    and v.sku = new.sku;

  if v_variation_id is not null then
    update variations set total_sales = total_sales + new.quantity where id = v_variation_id;
  else
    update products set total_sales = total_sales + new.quantity
    where slug = new.product_slug and brand_slug = v_brand_slug and sku = new.sku;
  end if;

  return new;
end;
$$;
```

- `v_` prefix on local variables avoids ambiguity with column names of the same name in PL/pgSQL.
- First tries to find a variation matching `(product_slug, brand_slug, sku)`. If found → increment `variations.total_sales`.
- If no variation matches → tries `products` directly (simple product with its own SKU).
- If neither matches (product deleted after checkout) → silent no-op. Merchant handles the refund manually.
- No `coalesce` needed because both `total_sales` columns are initialized to `0`.

---

## Supabase `.eq()` on Joined Tables

When you join a related table in Supabase and filter on its columns:

```ts
supabase.from("products").select("*, variations(*)").eq("variations.sku", "SKU-BLK")
```

This does **not** filter the joined rows — it acts as a binary filter on the parent row. If any variation matches, all variations are returned. If none match, the parent row is excluded entirely. It's an existence check, not a row filter.

To filter the joined rows themselves you need a separate query.

---

## User Endpoints → POST (not GET)

Cart, bookmarks, and orders are `POST` not `GET` even though they read data. Reason: they need `brandSlug` in the request body, and GET requests don't have a body. Sending query params would work but makes auth endpoints inconsistent with the rest of the pattern.

---

## DB Migrations

| File | Purpose |
|------|---------|
| `001_initial_schema.sql` | Source of truth — full schema from scratch |
| `002_user_cart_bookmarks.sql` | Historical — added cart_items and bookmarks tables |
| `003_orders.sql` | Historical — added orders, order_items, update_total_sales trigger |
| `drop_schema.sql` | Drops everything — used for local resets |

The live DB diverges from history as migrations are applied. `001` is always the authoritative reference for what the schema should look like.

---

## Tests

`tests/user.ts` — run with `npx tsx tests/user.ts`

Requires a fresh JWT (1 hour TTL) at the top of the file. Tests against the Vercel deployment.

Covers:
- Unauthorized access (no token → 401)
- Cart: POST get, PUT sync, GET after sync, PUT clear, GET after clear
- Bookmarks: same cycle
- Orders: POST get
- Validate-cart: valid item (200, exists, priceChanged false) + missing item (404, exists false)
- Checkout: invalid item (404) + valid item (200, `data.url` is string)
