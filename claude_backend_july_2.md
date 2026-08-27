# LEARNINGS (July 2, 2026)

Everything built and learned on the sunglass-server backend and its frontend integration.

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
| GET | `/api/public/filler` | no | n randomly shuffled products for a brand |
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

Using the admin client for user-scoped queries would bypass RLS — you'd have to manually add `.eq("user_id", user.id)` everywhere. The user client passes the JWT in headers so RLS handles it automatically.

### Response Shape

Every endpoint uses `ok()` and `err()` from `src/lib/api.ts`:

```ts
ok(data)               → { success: true,  data }                  // always 200
err(msg, status)       → { success: false, message }               // data undefined, dropped from JSON
err(msg, status, data) → { success: false, message, data }         // validate-cart + checkout only
```

- `ok()` always responds with 200 — status was hardcoded and the param removed.
- `err()` uses `message` (not `error`) as the field name.
- The optional `data` on errors is only used by validate-cart and checkout on 404/409/422 to return the per-item validation array alongside the message.
- `undefined` fields are dropped from JSON by `JSON.stringify` automatically — no spread or conditional needed.
- Single resource → `data` is an object
- Multiple resources → `data` is an array
- Paginated → `data` is an object with an array field + metadata: `{ products, page, totalPages, ... }`

### Env Vars

`NEXT_PUBLIC_` prefix is only needed for env vars used in client components. Since both Supabase clients are server-only (route handlers), the URL and anon key don't need the prefix:

```
SUPABASE_URL=...
SUPABASE_ANON_KEY=...       // safe to expose publicly by design, but kept private since server-only
SUPABASE_SERVICE_ROLE_KEY=... // never expose — bypasses RLS
```

### Auth Flow

1. User logs in via Supabase Auth. Supabase sets a cookie with the JWT.
2. Frontend reads the JWT and sends it as `Authorization: Bearer <token>`.
3. `createUserClient` extracts the token, initializes a Supabase client with it, calls `auth.getUser()` to verify, and returns `{ supabase, user }`.
4. RLS policies on `cart_items`, `bookmarks`, `orders`, `order_items` automatically filter to `auth.uid() = user_id`.
5. JWT is valid for 1 hour.

`auth.getUser()` is a method on the client — the client must exist before you can call it. You can't verify a token without first instantiating the client.

```ts
const { data: { user } } = await supabase.auth.getUser();
```

`data:` is the key to look up, `{ user }` is what to bind from its value. `data` is not available as a variable — only `user` is. On an invalid token, `data.user` is `null`. The `if (!user) return null` check covers all failure cases — no need to destructure `error`.

### Webhook Responses

The webhook endpoint is called by Stripe's servers, not the browser. The response just needs to be 200 so Stripe knows it was received. No client ever reads it.

---

## Error Handling

### Backend

Every endpoint validates required params first and returns `400` if missing. Auth is checked before params on user endpoints — no point validating input for an unauthenticated request.

Empty `items` arrays are a `400` on validate-cart and checkout — calling them with no items is a caller mistake, same as a missing param. Cart and bookmark PUT endpoints accept empty arrays — that's the clear operation.

Every DB query checks `error` and returns `500` on failure. Without this, a failed query returns `null` data silently.

`/api/public/item` uses `.single()` which treats zero rows as `PGRST116`. The code checks the error code to distinguish not-found from DB failure:

```ts
if (error?.code === "PGRST116") return err("Product not found", 404);
if (error) return err("Failed to fetch product", 500);
```

All other endpoints use `.select()` — zero rows returns `[]` with no error. Wrong `brandSlug` returns an empty array with `200`. Categories can also return `[]` if a valid brand has no categories — not an error.

### Frontend (actions.ts)

Three fetch helpers, each returning a `Response` (never null):

```ts
apiFetch(path, params)           // GET with next: { revalidate: 60 } cache
publicPostFetch(path, body)      // POST, no cache
authedFetch(path, method, body)  // POST/PUT with Bearer token
```

`authedFetch` pre-checks the user via `getUser()` before making a network call. If no session, returns a synthetic 401 immediately — avoids a round trip to the server.

All three wrap `fetch` in try/catch and return a synthetic `503 Response` on network failure.

Error handling per status:
- `401` → `redirect("/sign-in")`
- `404` on item → `notFound()`
- `404`/`409`/`422` on validate-cart and checkout → return structured `{ data, status }` for per-item UI
- `500`/`503` → `redirect("/try-again")`
- `400` should never appear in production (developer error)

### ApiResponse Generic Type

```ts
type ApiResponse<T, E = never> =
  | { success: true; data: T }
  | { success: false; message: string; data?: E };
```

Two type params let TypeScript know `json.data` is `ValidateCartItem[]` on error for validate-cart and checkout, while all other endpoints only have a success data type.

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

`Array.isArray()` is a type guard — ensures the value is iterable before calling `.map()`. Not sanitization.

SQL injection is not a concern — Supabase's query builder uses parameterized queries. Values passed to `.eq()`, `.in()`, `.ilike()` are never interpolated into raw SQL.

---

## Filter Map Pattern

```ts
const FILTER_MAP: Record<string, { sale?: boolean; minPrice?: number; maxPrice?: number }> = {
  "under-15": { maxPrice: 1500 },
  "15-25":    { minPrice: 1500, maxPrice: 2500 },
  "25-plus":  { minPrice: 2500 },
  "sale":     { sale: true },
};

const activeFilter = filterSlug ? FILTER_MAP[filterSlug] : undefined;
```

The map acts as an allowlist — any value not in it returns `undefined` and no filter is applied. Unknown filter slugs are silently ignored. `params.get()` always returns string or null, never a number.

Conditions use `!== undefined` not truthiness:

```ts
if (activeFilter?.minPrice !== undefined) q = q.gte("min_price_cents", activeFilter.minPrice);
```

This allows `0` as a valid filter value. Truthy checks would skip a `0`. `?.` makes the check safe when `activeFilter` is `undefined` — short-circuits to `undefined` instead of throwing.

---

## TypeScript: Optional Parameters

Putting optional params last avoids overloads:

```ts
// Clean — no overloads needed
err(message: string, status: number, data?: unknown)

// Requires overloads — TypeScript can't disambiguate without checking typeof
err(message: string, data: unknown, status: number)
```

Overloads work when types differ (object vs number), but keeping optional params last is simpler.

---

## Filler Endpoint

`GET /api/public/filler` returns `n` randomly shuffled products for a brand. No category, filters, or pagination.

Fetches `2n` from the DB then shuffles in JS and slices to `n`:

```ts
const shuffled = products.sort(() => Math.random() - 0.5).slice(0, n);
```

`Math.random() - 0.5` produces a value between -0.5 and 0.5 — roughly half the time negative (swap), half the time positive (don't swap). Fetching `2n` gives enough variety without over-fetching. If fewer than `n` products exist, all of them are returned.

Color swatches use the same deduplication as `/products` — each variation's `attribute` array is searched for the entry where `name === "color"`. Only one swatch per unique color slug is kept.

---

## Cart and Bookmark Sync

Cart and bookmarks are full replaces (delete + insert), not incremental. Running the same sync twice lands in the exact same state — idempotent by design.

If the delete succeeds but the insert fails (500), the cart is empty until the next sync. This is fine because local storage is the source of truth — any subsequent action re-syncs and restores the correct state.

Empty array PUT is the clear operation — deletes all rows and skips the insert. The `if (items.length > 0)` guard before the insert is clarity, not protection — `insert([])` is a no-op anyway. `[].map(...)` returns `[]` with no iterations and no errors.

---

## Categories and ISR Caching

The frontend fetches categories in a server component at layout level. Next.js `fetch` with `next: { revalidate: 60 }` caches the response at the CDN edge. After the first request, subsequent renders return near-instantly — effectively O(1).

The product page resolves a category slug to a category ID by re-fetching categories (hitting the cache), not by storing the tree in client state. Context can't be used in server components, and useEffect-based state misses the initial SSR render which hurts SEO. The double fetch is negligible because the second call hits the edge cache.

---

## Supabase Query Patterns

### `.eq()` on joined tables

```ts
supabase.from("products").select("*, variations(*)").eq("variations.sku", "SKU-BLK")
```

Does **not** filter the joined rows — it's a binary filter on the parent row. If any variation matches, all variations are returned. If none match, the parent row is excluded. It's an existence check, not a row filter.

### `.single()` vs `.select()`

- `.select()` always returns an array. Zero rows = `[]`, no error.
- `.single()` unwraps to one object. Zero rows = `PGRST116` error, `data` is `null`. More than one row = also an error.
- `.maybeSingle()` — zero rows = `data: null, error: null`. Use when absence is expected.

### Query builder pattern

Queries are built lazily — nothing runs until `await`. Conditions chain and return a new query object each time:

```ts
let q = supabase.from("products").select(...)
if (filter?.sale) q = q.eq("sale", true)
if (filter?.minPrice !== undefined) q = q.gte("min_price_cents", filter.minPrice)
const { data, error } = await q.range(from, to) // fires one SQL query
```

### Variations always return an array

Both `/products` and `/item` use `.map()` on variations, so the result is always `[]` or `[...]`, never `null`. The `product.variations ?? []` fallback handles the case where Supabase returns null for an empty relation before `.map()` is called.

### Name and image are always paired

Image `name` and `src` come from the same DB row — if one exists, the other does too. No need to handle them independently.

---

## Validate Cart + Checkout

### validate-cart

One DB query fetches all matching products and variations by brand + slug list. A `Map<"slug:sku", priceCents>` gives O(1) lookup per item. Returns per-item `{ exists, priceCents, priceChanged }`.

`priceChanged` short-circuits on missing items:

```ts
priceChanged: dbPrice !== null && dbPrice !== item.priceCents
```

If `dbPrice` is `null` (item doesn't exist), `priceChanged` is `false` — `exists: false` already signals the problem.

| Condition | Status |
|-----------|--------|
| all good | 200 |
| missing items only | 404 |
| price changed only | 409 |
| both | 422 |

On 404/409/422 the response is `{ success: false, message, data: [...] }`. On 200 it's `{ success: true, data: [...] }`.

### Checkout

Same validation logic but the map stores full entry objects (`priceCents`, `name`, `attribute`, `imageSrc`) because Stripe line items need all of it. Validate-cart only needs `priceCents`. Logic can't be shared because the map value types and DB selects differ.

### Composite key pattern

`"productSlug:sku"` as the map key avoids nested Maps. Works because `(brand_slug, slug)` is unique on products and `sku` is unique within a product.

### Checkout idempotency

`idempotencyKey = userId:hash(cart):orderCount`. Hash is SHA-256 of the canonicalized cart (sorted by SKU, using DB prices). Same cart + no new orders → same Stripe session. Any change breaks the key.

### Price enforcement

Prices in the Stripe session come from the DB. Frontend `priceCents` is only used for comparison during validation.

---

## Webhook

`POST /api/webhooks/stripe` handles `checkout.session.completed`.

1. Verify `stripe-signature` with `stripe.webhooks.constructEvent`.
2. Check if `orders` row with this `stripe_session_id` already exists — if yes, return 200 (idempotent).
3. Fetch expanded line items from Stripe.
4. Insert `orders` row: status = `"processing"`, shipping address from `session.collected_information.shipping_details`.
5. Insert `order_items` rows — one per line item.
6. If order_items insert fails, delete the orders row and return 500.

Name encoded as `"Name — Attribute"` at checkout creation. Webhook splits on `" — "` to recover name and attribute. `slugify(name)` reconstructs `product_slug`.

---

## Total Sales Trigger

A Postgres `AFTER INSERT` trigger on `order_items` increments `total_sales`.

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

- `v_` prefix avoids ambiguity with column names in PL/pgSQL.
- Tries variation first, falls back to simple product, silent no-op if neither found.
- No `coalesce` needed — both `total_sales` columns initialize to `0`.

---

## Schema Decisions

### products

- `sku text` — nullable. Only set if no variations (simple product).
- `total_sales int` — initialized to 0, incremented by trigger.
- `min_price_cents` / `max_price_cents` — denormalized for fast list queries.
- `attributes jsonb` — `[{ name, options: [{ option, slug, value? }] }]`. Display labels and hex values. Separate from `variations`.

### variations

- `attribute jsonb` — `[{ name, slug }]`. References slugs in `products.attributes`. No display labels — look them up from the parent.
- `total_sales int not null` — initialized to 0.

### order_items

- `attribute text` — nullable. Display string (e.g. `"Gloss Black / Standard"`). `null` for simple products.

### orders

- `status text not null` — no default. Always set explicitly to `"processing"` by the webhook.
- `shippingAddress` — always collected by Stripe at checkout, but typed as nullable on the frontend as a safety measure.

### cart_items

- `attribute jsonb not null` — `[{ name, option, slug }]`. Stores display labels and URL slugs so the frontend can display and link without a lookup.

---

## User Endpoints → POST (not GET)

Cart, bookmarks, and orders are `POST` not `GET` even though they read data. Reason: they need `brandSlug` in the body, and GET requests don't have a body. POST with a JSON body is cleaner for authenticated endpoints.

---

## DB Migrations

| File | Purpose |
|------|---------|
| `001_initial_schema.sql` | Source of truth — full schema from scratch |
| `002_user_cart_bookmarks.sql` | Historical — added cart_items and bookmarks |
| `003_orders.sql` | Historical — added orders, order_items, trigger |
| `drop_schema.sql` | Drops everything — dev only |

`001` is always the authoritative reference. Numbered migrations are historical — applied once in order.
