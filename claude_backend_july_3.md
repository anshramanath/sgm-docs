# LEARNINGS (July 3, 2026)

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
| POST | `/api/user/orders` | yes | get order history with refund info |
| POST | `/api/user/checkout` | yes | create Stripe checkout session |
| POST | `/api/webhooks/stripe` | stripe sig | handle checkout + refund events |

---

## Architecture

### Two Supabase Clients

```ts
createAdminClient()   // service role key — bypasses RLS, used in public endpoints and webhook
createUserClient(req) // anon key + user's JWT — respects RLS, returns { supabase, user } or null
```

Public endpoints use the admin client to read catalog data without needing a user token. User endpoints use the user client so RLS automatically scopes every query to the logged-in user. The webhook uses the admin client because Stripe has no user token.

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
- The optional `data` on errors is only for validate-cart and checkout on 404/409/422 — returns the per-item validation array alongside the message.
- `undefined` fields are dropped from JSON by `JSON.stringify` — no spread or conditional needed.

### Env Vars

`NEXT_PUBLIC_` prefix is only needed for client components. Since both Supabase clients are server-only, the URL and anon key don't need the prefix:

```
SUPABASE_URL=...
SUPABASE_ANON_KEY=...         // safe to expose publicly by design, kept private since server-only
SUPABASE_SERVICE_ROLE_KEY=... // never expose — bypasses RLS
```

### Auth Flow

1. User logs in via Supabase Auth. Supabase sets a cookie with the JWT.
2. Frontend reads the JWT and sends it as `Authorization: Bearer <token>`.
3. `createUserClient` extracts the token, initializes a Supabase client with it, calls `auth.getUser()` to verify, and returns `{ supabase, user }`.
4. RLS policies on `cart_items`, `bookmarks`, `orders`, `order_items` automatically filter to `auth.uid() = user_id`.
5. JWT is valid for 1 hour.

`auth.getUser()` is a method on the client — the client must exist before you can call it. You cannot verify a token without first instantiating the client.

```ts
const { data: { user } } = await supabase.auth.getUser();
```

`data:` is the key to look up, `{ user }` is what to bind from its value. `data` is not available as a variable — only `user` is. On an invalid token, `data.user` is `null`. `if (!user) return null` covers all failure cases — no need to destructure `error`.

### Webhook Events

The webhook uses a switch so new events can be added cleanly:

```ts
switch (event.type) {
  case "checkout.session.completed": // create order + order_items
  case "charge.refunded":            // update refunded_cents + status
  default:                           // return 200, ignore
}
```

`charge.refunded` fires when a refund is issued from the Stripe dashboard. `charge.amount` is the original charge, `charge.amount_refunded` is the cumulative refunded amount. Comparing them determines full vs partial refund.

`payment_intent` on a Stripe object can be either a string ID or an expanded `PaymentIntent` object — the ternary `typeof charge.payment_intent === "string" ? charge.payment_intent : charge.payment_intent?.id` normalizes it to a string ID either way. In practice it's always a string when unexpanded.

Webhook responses are never seen by the client — Stripe only needs a 200 to confirm receipt.

---

## Error Handling

### Backend

Auth is checked before params on user endpoints — no point validating input for an unauthenticated request.

Empty `items` arrays are a `400` on validate-cart and checkout — calling them with no items is a caller mistake. Cart and bookmark PUT endpoints accept empty arrays — that's the clear operation.

Every DB query checks `error` and returns `500` on failure. Without this, a failed query returns `null` data silently.

`/api/public/item` uses `.single()` which treats zero rows as `PGRST116`:

```ts
if (error?.code === "PGRST116") return err("Product not found", 404);
if (error) return err("Failed to fetch product", 500);
```

All other endpoints use `.select()` — zero rows returns `[]` with no error. Wrong `brandSlug` returns an empty array with 200. Categories can return `[]` if a valid brand has no categories — not an error.

### Frontend (actions.ts)

Three fetch helpers, each returning a `Response` (never null):

```ts
apiFetch(path, params)           // GET with next: { revalidate: 60 } cache
publicPostFetch(path, body)      // POST, no cache
authedFetch(path, method, body)  // POST/PUT with Bearer token
```

`authedFetch` pre-checks the user via `getUser()` before making a network call — returns a synthetic 401 immediately if no session, avoiding a round trip.

All three wrap `fetch` in try/catch and return a synthetic `503 Response` on network failure.

Error handling per status:
- `401` → `redirect("/sign-in")`
- `404` on item → `notFound()`
- `404`/`409`/`422` on validate-cart and checkout → return structured `{ data, status }` for per-item UI
- `500`/`503` → `redirect("/try-again")`
- `400` should never appear in production

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
```

The map acts as an allowlist — any value not in it returns `undefined` and no filter is applied. `params.get()` always returns string or null, never a number.

Conditions use `!== undefined` not truthiness to allow `0` as a valid filter value. `?.` makes the check safe when `activeFilter` is `undefined`.

---

## TypeScript: Optional Parameters

Putting optional params last avoids overloads:

```ts
err(message: string, status: number, data?: unknown)  // clean
err(message: string, data: unknown, status: number)    // requires overloads
```

Overloads work when types differ (object vs number), but keeping optional params last is simpler.

---

## Filler Endpoint

`GET /api/public/filler?brandSlug=...&n=20` — returns `n` randomly shuffled products. No category, filters, or pagination.

Fetches `2n` from the DB then shuffles and slices:

```ts
const shuffled = products.sort(() => Math.random() - 0.5).slice(0, n);
```

`Math.random() - 0.5` produces a value between -0.5 and 0.5 — roughly half the time negative (swap), half the time positive (don't swap). `n` is clamped to 1–100. If fewer than `n` products exist, all are returned.

Color swatches use the same deduplication as `/products` — only one swatch per unique color slug is kept. `color` is one entry in the variation's `attribute` array where `name === "color"`.

---

## Cart and Bookmark Sync

Cart and bookmarks are full replaces (delete + insert) — idempotent by design. Local storage is the source of truth. A 500 on insert leaves the DB empty until the next sync, which restores the correct state.

Empty array PUT is the clear operation. `if (items.length > 0)` before the insert is clarity, not protection — `insert([])` is a no-op and `[].map(...)` returns `[]` with no iterations.

---

## Refunds

`refunded_cents` is `null` on a new order (no refund), set to the cumulative amount on `charge.refunded`. `null` means no refund, a value means some or all was refunded. Status values:

| Status | Meaning |
|--------|---------|
| `processing` | Order received, being prepared |
| `shipped` | In transit |
| `delivered` | Arrived |
| `refunded` | Fully refunded |
| `partially_refunded` | Partially refunded |

---

## Categories and ISR Caching

`next: { revalidate: 60 }` caches the response at the CDN edge — subsequent renders return near-instantly. The product page re-fetches categories to resolve a slug to an ID (hitting the cache), rather than storing the tree in client state. Context can't be used in server components, and useEffect-based state misses the initial SSR render which hurts SEO.

---

## Supabase Query Patterns

### `.eq()` on joined tables

Filters the parent row by whether a matching child exists — does not filter the child rows themselves. If any child matches, all children are returned.

### `.single()` vs `.select()`

- `.select()` — zero rows = `[]`, no error.
- `.single()` — zero rows = `PGRST116` error. More than one row = also an error.
- `.maybeSingle()` — zero rows = `data: null, error: null`.

### Query builder pattern

Queries are built lazily — nothing runs until `await`. Conditions chain and return a new query object, allowing conditional query building before a single SQL query fires.

### Variations always return an array

`.map()` on variations always produces `[]` or `[...]`, never `null`. `product.variations ?? []` handles the case where Supabase returns null for an empty relation.

### Name and image are always paired

Image `name` and `src` come from the same DB row — if one exists, the other does too.

---

## Validate Cart + Checkout

### validate-cart

One DB query fetches all matching products and variations. A `Map<"slug:sku", priceCents>` gives O(1) lookup per item.

`priceChanged` short-circuits on missing items:
```ts
priceChanged: dbPrice !== null && dbPrice !== item.priceCents
```

| Condition | Status |
|-----------|--------|
| all good | 200 |
| missing items only | 404 |
| price changed only | 409 |
| both | 422 |

### Checkout

Same validation logic but the map stores full entry objects (`priceCents`, `name`, `attribute`, `imageSrc`) for Stripe line items. Can't share logic with validate-cart because map value types and DB selects differ.

`"productSlug:sku"` as the composite map key avoids nested Maps.

`idempotencyKey = userId:hash(cart):orderCount` — same cart + no new orders → same Stripe session.

Prices in the Stripe session come from the DB — frontend `priceCents` is only used for validation comparison.

---

## Webhook

`checkout.session.completed`:
1. Verify `stripe-signature`.
2. Check `stripe_session_id` uniqueness — if exists, return 200 (idempotent).
3. Insert `orders` row with status `"processing"`, `refunded_cents: null`.
4. Insert `order_items` — one per line item.
5. If order_items insert fails, delete the order and return 500.

Name encoded as `"Name — Attribute"` at checkout. Webhook splits on `" — "` to recover name and attribute. `slugify(name)` reconstructs `product_slug`.

`charge.refunded`:
1. Normalize `payment_intent` to string ID.
2. Update `refunded_cents = charge.amount_refunded`.
3. Set status to `"refunded"` if `amount_refunded === amount`, else `"partially_refunded"`.

---

## Total Sales Trigger

Postgres `AFTER INSERT` trigger on `order_items`. Tries variation first, falls back to simple product, silent no-op if neither found. `v_` prefix on local variables avoids ambiguity with column names in PL/pgSQL. No `coalesce` needed — both `total_sales` columns initialize to `0`.

---

## Schema Decisions

### products
- `sku text` — nullable, only set for simple products (no variations).
- `total_sales int` — initialized to 0, incremented by trigger.
- `min_price_cents` / `max_price_cents` — denormalized for fast list queries.
- `attributes jsonb` — display labels and hex values for all attribute types.

### variations
- `attribute jsonb` — `[{ name, slug }]` — references slugs in `products.attributes`. No display labels stored.
- `total_sales int not null` — initialized to 0.

### orders
- `refunded_cents int` — nullable. `null` = no refund, value = cumulative refunded amount.
- `status text not null` — always set explicitly, no default.
- `stripe_payment_intent text not null unique` — used to match `charge.refunded` events.

### order_items
- `attribute text` — nullable. Display string (e.g. `"Gloss Black / Standard"`). `null` for simple products.

### cart_items
- `attribute jsonb not null` — `[{ name, option, slug }]`. Stores display labels and URL slugs.

---

## User Endpoints → POST (not GET)

Cart, bookmarks, and orders are `POST` not `GET` because they need `brandSlug` in the body. GET requests don't have a body. POST with a JSON body is cleaner for authenticated endpoints.

---

## DB Migrations

| File | Purpose |
|------|---------|
| `001_initial_schema.sql` | Source of truth — full schema from scratch |
| `002_user_cart_bookmarks.sql` | Historical — added cart_items and bookmarks |
| `003_orders.sql` | Historical — added orders, order_items, trigger |
| `drop_schema.sql` | Drops everything — dev only |

`001` is always the authoritative reference. Numbered migrations are historical — applied once in order.
