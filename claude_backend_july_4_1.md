# Changelog (July 4, 2026)

Everything built and decided since the last handoff doc.

---

## API Shape

### `ok()` and `err()` signatures

```ts
ok(data: unknown)                              // always 200
err(message: string, status: number, data?: unknown)  // optional structured payload
```

Response shapes:
```json
{ "success": true,  "data": { ... } }
{ "success": false, "message": "..." }
{ "success": false, "message": "...", "data": { ... } }
```

The third shape is only used by `/validate-cart` and `/checkout` on 404/409/422 — `data` contains the per-item validation array.

`err()` puts `data` last as an optional param. Order matters for TypeScript to distinguish overloads without needing them — `string` and `number` are different types so the two required params are unambiguous.

`undefined` fields are dropped by `JSON.stringify` automatically — no spread needed to omit optional fields.

### Env vars

Supabase env vars do not use `NEXT_PUBLIC_` prefix — the admin and user clients are server-only files. `NEXT_PUBLIC_` is only needed for client components.

---

## Endpoints

### validate-cart

- Added DB error check — previously the Supabase query didn't destructure `error`, so a DB failure returned 200 with all items as not found.
- Empty `items` array now returns `400` instead of silently succeeding.
- On non-200 status returns `err("Cart validation failed", status, result)` — the validation array is in `data`.

### checkout

- Mirrors validate-cart: on validation failure returns `err("Cart validation failed", status, validation)` with the array in `data`.

### orders

- Auth moved before param parsing — no point reading the body if the user isn't logged in.
- Added `refunded_cents` to the select query and response mapping as `refundedCents`.
- `refundedCents` is `null` when no refund has been issued.

### products + sale

- Extracted result object before `return ok(result)` for readability.

### filler — `GET /api/public/filler`

New endpoint. Returns `n` randomly shuffled products for a brand with no category, filter, or pagination.

- Fetches `2n` from DB, shuffles with `Math.random() - 0.5`, slices to `n`.
- `n` defaults to 20, max 100.
- Same product shape as `/products` (image, variations deduped by color).
- Unknown `brandSlug` returns empty array with 200 — same as other list endpoints.

### views — `POST /api/public/views`

New endpoint. Atomically increments view count for a category or product.

- Accepts `brandSlug` + exactly one of `categoryId` or `productSlug`.
- Validation: if neither provided → 400; if both provided → 400.
- Uses `Record<string, string>` target object — only truthy body params get added as keys, so `Object.keys(target).length` gives the count cleanly.
- Switch on `keys[0]` for extensibility as more countable things get added.
- Uses Supabase RPC for atomic increment — the query builder cannot express `col = col + 1`. Two-step read+write has a race condition.
- Returns `ok(target)` — echoes back which target was incremented (e.g. `{ categoryId: "uuid" }`).

### stripe webhook

- Refactored to switch statement for extensibility.
- Added `charge.refunded` handler:
  - Matches order by `stripe_payment_intent` (normalised from string or expanded object via ternary).
  - `charge.amount` is the original charge total; `charge.amount_refunded` is cumulative.
  - Full refund: `amount_refunded === amount` → status `refunded`.
  - Partial refund: `amount_refunded < amount` → status `partially_refunded`.
  - Sets `refunded_cents` to `charge.amount_refunded`.
- `refunded_cents: null` added explicitly to the order insert for clarity (null = no refund).

---

## Database

### Schema additions

`refunded_cents int` added to `orders` — nullable, null means no refund issued.

`view_count int default null` added to `categories` — nullable because categories may never be viewed.

`view_count int not null default 0` added to `products`.

### RPC functions (in `001_initial_schema.sql`)

```sql
increment_category_view(p_id uuid, p_brand_slug text)
-- UPDATE categories SET view_count = coalesce(view_count, 0) + 1
-- WHERE id = p_id AND brand_slug = p_brand_slug

increment_product_view(p_slug text, p_brand_slug text)
-- UPDATE products SET view_count = view_count + 1
-- WHERE slug = p_slug AND brand_slug = p_brand_slug
```

`coalesce` needed for category because `view_count` starts null. Products start at 0 so no coalesce needed.

A single SQL `UPDATE ... SET col = col + 1` is atomic — Postgres evaluates the expression server-side. No trigger needed; a trigger would only help if writes came from multiple sources (e.g. application + direct DB writes). Since all increments go through the API, the RPC is sufficient.

---

## Key Decisions and Rules

**Switch statements need `break` or `return` to prevent fall-through.** Unlike if-else chains, JS switch cases fall through by default — after one case runs, execution continues into the next unless you exit. `return` works the same as `break` for preventing fall-through and is cleaner when each case produces a value.

**Supabase query builder can't express `col = col + 1`.** Any atomic increment needs a Postgres function called via `supabase.rpc()`. Two-step read+write in application code is not atomic — two concurrent requests can both read the same value and both write `value + 1`, net result `+1` instead of `+2`.

**Filter map as allowlist.** Unknown filter slugs resolve to `undefined` in the filter map and are silently ignored — no error returned, all products returned instead. This is intentional — graceful degradation for stale client filter params.

**`payment_intent` on Stripe objects can be a string or an expanded object.** Normalise with a ternary before use: `typeof x === "string" ? x : x?.id`.

**Empty string is falsy in JS.** Truthy checks on body params correctly reject empty strings — `if (!body.brandSlug)` handles both missing and empty.

**`undefined` fields are dropped by `JSON.stringify`.** No need to spread or conditionally build objects to omit optional fields — just set them to `undefined` or leave them out.

**`Record<string, string>` is a typed object, not an array.** Use `Object.keys()` to get the keys as an array for length checks and iteration.

**`Math.random() - 0.5` for shuffle.** Produces a value in [-0.5, 0.5] — half the time negative (swap), half positive (don't). Not cryptographically uniform but fine for display shuffling.

**`coalesce` for nullable counters.** When a column can be null, `coalesce(col, 0) + 1` treats null as zero. Skip it when the column has a non-null default.

**Prematurely added `data` before `status` in `err()`** caused an overload mess — TypeScript couldn't distinguish signatures when types overlapped. Fixed by putting optional `data` last.

---

## Brand System (`src/lib/brand.ts`)

Central brand registry for all three brands:

| Brand | Slug | Accent |
|-------|------|--------|
| BikerShades | `bikershades` | `#C93A2B` |
| proSPORT Sunglasses | `prosport-sunglasses` | `#2EA3DC` |
| Sunglass Monster | `sunglass-monster` | `#E0522D` |

Each brand has: `name`, `shortName`, `slug`, `url`, `favicon`, `logo`, `hero`, `accent`, `heroCopy`, `categoryImages`, `editorial`, `announcements`.

`getBrand()` resolves the current brand from `NEXT_PUBLIC_BRAND_SLUG` env var — used by single-brand frontends.
`getBrands()` returns all brands sorted with the current brand first — used by the admin dashboard.

Public assets live at `/public/[slug]/`: `logo.jpg`, `hero.jpg`, `favicon.png`, `cat-1..5.jpg`, `edit-1..2.jpg`.

---

## Admin Dashboard

### Plan

- Built in this repo (Next.js App Router) alongside the API routes.
- Pages are scoped per brand — URL carries the brand slug.
- Admins table: `user_id uuid primary key references auth.users(id)` — any user in this table is an admin with access to all brands.
- No stock/inventory concept — products don't have stock status. Ignored.

### Route structure

```
src/app/admin/
  layout.tsx                ← auth guard: checks admins table, redirects if not admin
  page.tsx                  ← redirects to /admin/[firstBrand]/overview
  [brandSlug]/
    layout.tsx              ← sidebar shell, brand config
    overview/page.tsx
    orders/page.tsx
    products/
      page.tsx
      [productSlug]/page.tsx
    categories/page.tsx
    analytics/page.tsx
  _components/
    sidebar.tsx
    nav.tsx
    brand-switcher.tsx
```

Sign-in is a separate page outside `/admin` — the admin layout assumes authentication and redirects if not present.

### Pages

**Overview** — catalogue stats (products, on-sale count, featured count, category count) + order stats (active orders, completed orders, revenue) + recently added products.

**Orders** — all orders for the brand, filterable by status. Expandable rows show shipping address, Stripe payment intent ID, and line items. Statuses: `processing`, `shipped`, `delivered`, `refunded`, `partially_refunded`.

**Products** — full catalogue list with search and category filter. Clicking a product opens the detail page.

**Product detail** — editable: name, slug, SKU, description, summary bullets, categories, featured flag, product images (with sort order). Variations section: each variation has SKU, regular price, sale price, sale flag, attributes, and variation images (with sort order). Variations can be added and removed.

**Categories** — hierarchical tree showing name, slug, sort order, product count, view count. Read-only for now.

**Analytics** — top products by views, top categories by views, top products by sales (units sold).

### Design system

Uses the Aritzia design system tokens via CSS variables. Each brand's accent color is applied to active nav items, brand switcher selection, sale prices, and refund status indicators. The sidebar shows the active brand's logo.

### Prototype (admin_dashboard.dc.html)

Fully interactive DC prototype with real state management. Covers all pages including product detail with mutations. Uses placeholder asset paths (`assets/`) — real paths use `/[brandSlug]/logo.jpg` convention from `brand.ts`.
