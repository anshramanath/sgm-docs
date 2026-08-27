# LEARNINGS (June 25, 2026)

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
ok(data)               → { success: true,  data }
err(msg, status)       → { success: false, message, data: undefined } // data dropped from JSON
err(msg, status, data) → { success: false, message, data }            // validate-cart + checkout only
```

- `ok()` always responds with 200 — the status param was removed because it was never anything else.
- `err()` uses `message` (not `error`) as the field name.
- The optional `data` field on errors is only used by validate-cart and checkout on 404/409/422 to return the per-item validation array alongside the error message.
- `undefined` fields are dropped from JSON automatically by `JSON.stringify` — no need to spread or conditionally include them.
- Single resource → `data` is an object: `{ "url": "..." }`
- Multiple resources → `data` is an array: `[{ ... }, { ... }]`
- Paginated → `data` is an object with an array field + metadata: `{ products: [...], page, totalPages, ... }`

### Auth Flow

1. User logs in via Supabase Auth (email/password). Supabase sets a cookie with the JWT.
2. Frontend reads the JWT from the cookie and sends it as `Authorization: Bearer <token>`.
3. `createUserClient` extracts the token, initializes a Supabase client with it, calls `auth.getUser()` to verify it, and returns `{ supabase, user }`.
4. RLS policies on `cart_items`, `bookmarks`, `orders`, and `order_items` automatically filter to `auth.uid() = user_id`.
5. JWT is valid for 1 hour.

---

## Error Handling

### Backend

Every endpoint validates required params and returns `400` if missing. These are developer errors — the frontend should never hit them in production. Empty `items` arrays are also a `400` — calling validate-cart or checkout with no items is a caller mistake, same as a missing param.

Every DB query checks `error` and returns `500` on failure. Without this check, a failed query returns `null` data with no error signal — the caller silently gets an empty array or missing fields.

`/api/public/item` uses `.single()` which treats zero rows as an error (`PGRST116`). The error code is checked to distinguish "not found" (404) from a real DB failure (500):

```ts
if (error?.code === "PGRST116") return err("Product not found", 404);
if (error) return err("Failed to fetch product", 500);
```

All other endpoints use regular `.select()` which always returns an array — zero rows is `[]` with no error. An incorrect `brandSlug` returns an empty array with `200`, not an error.

### Frontend

```ts
const res = await apiFetch(path, params);
if (!res.ok) // covers 4xx, 5xx, and synthetic 503 network failure
const json = await res.json(); // happy path
```

`apiFetch` wraps `fetch` in a try/catch and returns a synthetic `503 Response` on network failure, so the caller always gets a `Response`. `!res.ok` covers everything: network failure, bad params, DB errors. A `400` is treated the same as a `500` on the frontend — both show a generic error.

For validate-cart and checkout, `!res.ok` on 404/409/422 means there's a `json.data` array with per-item results alongside `json.message`.

### Status Code Summary

| Status | Cause |
|--------|-------|
| `200` | Success |
| `400` | Missing or invalid params (developer error) |
| `401` | Missing or invalid JWT |
| `404` | Item not found (`.single()` only) or cart item missing |
| `409` | Price changed |
| `422` | Missing items + price changed |
| `500` | DB or internal failure |
| `503` | Network failure (synthetic, client-side only) |

---

## Input Validation vs SQL Injection

`Array.isArray()` is a type guard — it ensures the value is iterable before calling `.map()` on it. Without it, `"items": "foo"` from a malicious request would throw when iterated. This is defensive typing, not sanitization.

SQL injection is not a concern because Supabase's query builder uses parameterized queries. Values passed to `.eq()`, `.in()`, `.ilike()` etc. are never interpolated into raw SQL. The only risk would be building raw SQL strings with user input, which this codebase doesn't do.

---

## Filter Map Pattern

```ts
const FILTER_MAP: Record<string, { minPrice?: number; maxPrice?: number }> = {
  "under-15": { maxPrice: 1500 },
  "15-25":    { minPrice: 1500, maxPrice: 2500 },
  "25-plus":  { minPrice: 2500 },
};

const activeFilter = FILTER_MAP[filterSlug ?? ""];
```

The map acts as an allowlist — any value not in it returns `undefined` and no filter is applied. Unknown or malformed filter slugs are silently ignored. `params.get()` always returns a string or `null`, never a number, so type coercion edge cases don't apply.

Conditions use `!== undefined` rather than truthiness:

```ts
if (activeFilter?.minPrice !== undefined) q = q.gte("min_price_cents", activeFilter.minPrice);
```

This allows `0` as a valid filter value. Truthy checks (`if (activeFilter?.minPrice)`) would skip a `0`. Not relevant to current filters (which start at 1500), but the correct habit in case the map changes.

`?.` (optional chaining) is safe property access — `activeFilter?.minPrice` returns `undefined` if `activeFilter` is `undefined` instead of throwing. It's what makes the check safe when no filter is active.

---

## TypeScript: Optional Parameters

`data?: unknown` means the parameter is optional — callers can omit it. Without `?`, TypeScript requires it at every call site.

When optional params are the same type as required ones, TypeScript can't disambiguate them without overloads. Putting the optional param last avoids this:

```ts
// Clean — no overloads needed, status is always a number in position 2
err(message: string, status: number, data?: unknown)

// Requires overloads — data and status could both be unknown in position 2
err(message: string, data: unknown, status: number)
```

The version with overloads works only when the types differ (e.g. `data` is an object, `status` is a number), but overloads add noise. Keeping optional params last is simpler.

---

## Schema Decisions

### products

- `sku text` — nullable. Only set if the product has no variations (simple product). Variations carry their own SKUs.
- `total_sales int` — initialized to 0, incremented by the `update_total_sales` trigger.
- `min_price_cents` / `max_price_cents` — denormalized from variations for fast list queries.
- `attributes jsonb` — array of `{ name, options: [{ option, slug, value? }] }`. Defines what attributes (color, size, etc.) the product has, with display labels and color hex values. Separate from `variations`.

### variations

- `attribute jsonb` — array of `{ name, slug }` pairs. References slugs in `products.attributes`. No display labels stored — use the parent product's `attributes` to look them up.
- `total_sales int not null` — always initialized to 0.

### order_items

- `attribute text` — nullable. Display string for variation products (e.g. `"Gloss Black / Standard"`). `null` for simple products. Stored as a string because it's display-only; no need to parse it after the fact.

### orders

- `status text not null` — no default. Application always sets it explicitly to `"processing"`.

### cart_items

- `attribute jsonb not null` — array of `{ name, option, slug }`. Stores display labels (`option`) and URL slugs (`slug`) so the frontend can both display the selection and link to the product page without a lookup.
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

On 404/409/422 the response is `{ success: false, message, data: [...] }` — the validation array is in `data` alongside the error message. On 200 it's `{ success: true, data: [...] }`.

Checkout runs the same validation before creating a Stripe session.

### Composite key pattern

`"productSlug:sku"` is used as the Map key throughout validate-cart and checkout. This avoids nested Maps and gives O(1) lookup. It works because `(brand_slug, slug)` is unique on products and `sku` is unique within a product.

### Why the Map has extra entries

The query fetches all variations for each matching product, not just the ones in the cart. So the Map is larger than needed — but it's still one query, and the extra entries are ignored. The alternative (filtering to exact slug+sku pairs in SQL) requires two queries in the worst case (first check variations, then fall back to product SKU).

### Checkout idempotency

Stripe checkout sessions are created with an `idempotencyKey = userId:hash(cart):orderCount`. The hash is a SHA-256 of the canonicalized cart (sorted by SKU, using DB prices). If the user opens checkout twice with the same cart and no new orders, the same Stripe session is returned. Any change to products, prices, or a completed order breaks the key and creates a fresh session.

### Price enforcement

Prices in the checkout session come from the DB — the frontend `priceCents` is only used for comparison during validation. Stripe line items always use `entry.priceCents` from the server's entryMap.

---

## Webhook

`POST /api/webhooks/stripe` handles `checkout.session.completed`.

1. Verify `stripe-signature` header with `stripe.webhooks.constructEvent`.
2. Check if `orders` row with this `stripe_session_id` already exists — if yes, return 200 (idempotent).
3. Fetch expanded line items from Stripe.
4. Insert `orders` row: status = `"processing"`, shipping address from `session.collected_information.shipping_details`.
5. Insert `order_items` rows — one per Stripe line item.
6. If order_items insert fails, delete the orders row and return 500 (keeps DB consistent).

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

## Supabase Query Patterns

### `.eq()` on joined tables

When you join a related table in Supabase and filter on its columns:

```ts
supabase.from("products").select("*, variations(*)").eq("variations.sku", "SKU-BLK")
```

This does **not** filter the joined rows — it acts as a binary filter on the parent row. If any variation matches, all variations are returned. If none match, the parent row is excluded entirely. It's an existence check, not a row filter. To filter the joined rows themselves you need a separate query.

### `.single()` vs `.select()`

- `.select()` always returns an array. Zero rows = `[]`, no error.
- `.single()` unwraps to one object. Zero rows = `PGRST116` error, `data` is `null`. More than one row = also an error.
- `.maybeSingle()` — zero rows = `data: null, error: null`. One row = object. Use when absence is expected.

### Query builder pattern

Supabase queries are built lazily — nothing runs until `await`. Each `.eq()`, `.gte()`, `.lte()` appends a condition and returns a new query object. This allows conditional query building:

```ts
let q = supabase.from("products").select(...)
if (filter?.sale) q = q.eq("sale", true)
if (filter?.minPrice !== undefined) q = q.gte("min_price_cents", filter.minPrice)
const { data, error } = await q.range(from, to) // fires one SQL query
```

---

## User Endpoints → POST (not GET)

Cart, bookmarks, and orders are `POST` not `GET` even though they read data. Reason: they need `brandSlug` in the request body, and GET requests don't have a body. Sending it as a query param would expose it in logs and URLs unnecessarily — POST with a JSON body is cleaner for authenticated endpoints.

---

## DB Migrations

| File | Purpose |
|------|---------|
| `001_initial_schema.sql` | Source of truth — full schema from scratch |
| `002_user_cart_bookmarks.sql` | Historical — added cart_items and bookmarks tables |
| `003_orders.sql` | Historical — added orders, order_items, update_total_sales trigger |
| `drop_schema.sql` | Drops everything — used for local resets |

`001` is always the authoritative reference for what the schema should look like. The numbered migrations are historical — applied once to the live DB in order.

---

## Tests

`tests/user.ts` — run with `npx tsx tests/user.ts`

Requires a fresh JWT (1 hour TTL) and brand slug at the top of the file. Tests against the Vercel deployment.

Covers:
- Unauthorized access (no token → 401)
- Cart: POST get, PUT sync, GET after sync, PUT clear, GET after clear
- Bookmarks: same cycle
- Orders: POST get
- Validate-cart: valid item (200, exists, priceChanged false) + missing item (404, exists false)
- Checkout: invalid item (404) + valid item (200, `data.url` is string)
