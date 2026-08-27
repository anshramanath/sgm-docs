# BikerShades Server — Project Documentation (May 29, 2026)

Everything built, every decision made, and every gotcha encountered.

---

## What Was Built

A Next.js 16 app serving as the central backend for the BikerShades e-commerce brand. It does three things:

1. **Admin dashboard** — web UI for managing 589 products, 5,784 variations, 63 categories, and 2,646 images
2. **API server** — all brand frontends call this app's routes; they never touch Supabase directly
3. **MCP server** — planned; exposes catalog tools for AI agent access

---

## Architecture Decisions

### Why a central API server instead of letting frontends hit Supabase directly

Supabase's `service_role` key bypasses all Row Level Security. If frontends had it, a leak would expose the entire database. Instead:
- This app holds the `service_role` key in server-side env vars only
- Frontends use the anon key for their own auth, but catalog data comes through this app's API routes
- RLS is still enabled on all tables as a safety net against accidental direct access

### Why Supabase SSR auth instead of NextAuth

Simpler setup — auth cookies are managed by `@supabase/ssr`, session refresh happens in the proxy middleware, and admin gating is a single table check (`admins`). NextAuth would have added a credentials provider, JWT configuration, and a separate session store for no real gain here.

### Why two Supabase clients

| Client | File | Key | Use |
|---|---|---|---|
| Admin | `lib/supabase/server.ts` | `service_role` | All API routes and server components — bypasses RLS |
| Session | `lib/supabase/session.ts` | `anon` | Reading the logged-in user's identity from their auth cookie |
| Browser | `lib/supabase/client.ts` | `anon` | Login form only — signs the user in on the client |

The session client uses the anon key but reads the user's JWT from the cookie — it cannot do privileged operations.

### Why `requireAdmin()` as a shared guard

Every write route (`POST`, `PUT`, `DELETE`) calls `requireAdmin()` before touching the database. It does two things in sequence:
1. Reads the session cookie → gets the user
2. Checks the `admins` table for that `user_id`

A user can be authenticated via Supabase Auth but still get a 403 if they're not in `admins`. This means you can have Supabase Auth users who aren't admins (e.g. future customer accounts) without any risk of them accessing admin routes.

---

## File Structure

```
bikershades-server/
├── app/
│   ├── (auth)/
│   │   ├── layout.tsx
│   │   └── login/page.tsx
│   ├── (dashboard)/
│   │   ├── layout.tsx                        Sidebar + bg wrapper
│   │   ├── dashboard/page.tsx                Stats + recent products
│   │   ├── products/
│   │   │   ├── page.tsx                      Paginated list + filters
│   │   │   └── [id]/
│   │   │       ├── page.tsx                  Detail + edit form
│   │   │       └── variations/page.tsx       Full variation editor
│   │   ├── categories/page.tsx
│   │   └── flagged/page.tsx
│   ├── api/
│   │   ├── products/
│   │   │   ├── route.ts                      GET list, POST
│   │   │   └── [id]/
│   │   │       ├── route.ts                  GET single, PUT, DELETE
│   │   │       └── variations/route.ts       GET
│   │   ├── variations/[id]/route.ts          PUT, DELETE
│   │   ├── categories/
│   │   │   ├── route.ts                      GET, POST
│   │   │   └── [id]/route.ts                 DELETE
│   │   ├── images/
│   │   │   ├── upload/route.ts               POST (multipart → Storage)
│   │   │   ├── [id]/route.ts                 DELETE (DB + Storage)
│   │   │   └── reorder/route.ts              PUT (bulk sort_order)
│   │   └── admin/
│   │       ├── me/route.ts                   GET (session check)
│   │       └── users/
│   │           ├── route.ts                  POST (grant admin)
│   │           └── [userId]/route.ts         DELETE (revoke admin)
│   ├── page.tsx                              Redirects → /dashboard
│   ├── layout.tsx                            Root layout
│   └── globals.css
├── components/
│   ├── Sidebar.tsx
│   ├── ProductEditForm.tsx
│   ├── ProductFilters.tsx
│   ├── VariationRow.tsx
│   ├── CategoryManager.tsx
│   └── FlaggedProductImporter.tsx
├── lib/
│   ├── supabase/
│   │   ├── server.ts                         service_role client
│   │   ├── client.ts                         browser anon client
│   │   └── session.ts                        cookie-aware server client
│   └── auth.ts                               requireAdmin() guard
├── proxy.ts                                  Auth middleware (Next.js 16)
├── next.config.ts
├── .env.local
└── migration/
    └── reshaped/                             Flagged product JSON files
```

---

## Database Schema

Source of truth: `migration/db/001_initial_schema.sql`

```
brands              id, name
categories          id, brand_id, name
products            id, brand_id, name, slug, sku, wp_url, product_url,
                    description, summary text[], attributes jsonb,
                    sale, regular_price_cents, sale_price_cents,
                    average_rating, review_count, in_stock,
                    weight, weight_unit, length, width, height, dimension_unit
product_categories  product_id, category_id  (junction)
variations          id, product_id, slug, sku, variation text[],
                    wp_url, product_url, description,
                    sale, regular_price_cents, sale_price_cents,
                    in_stock, weight, weight_unit, length, width, height, dimension_unit
product_images      id, product_id, src, name, sort_order
variation_images    id, variation_id, src, name, sort_order
description_images  id, product_id?, variation_id?, src, name, sort_order
admins              user_id → auth.users(id)
```

### Key field notes

**`attributes` (jsonb on products)** — grouped attribute options for listing-page queries without joining variations:
```json
[
  { "name": "color", "terms": ["black-clear", "blue-smoke"] },
  { "name": "size",  "terms": ["small-medium", "large-xlarge"] }
]
```

**`variation text[]`** — the specific combo for one variation: `["black-clear", "small-medium"]`. Maps positionally to `attributes[].terms`.

**`summary text[]`** — parsed bullet points from a product's short description.

**`description_images`** — images extracted from description HTML. One of `product_id` or `variation_id` must be set (DB-enforced CHECK constraint).

**`wp_url`** — original bikershades.com URLs, kept for 301 redirects from Google-indexed pages.

---

## API Routes

All routes are server-side. Write routes require an admin session.

### Products

| Method | Route | Auth | Notes |
|---|---|---|---|
| GET | `/api/products` | — | Params: `brand_id`, `category_id`, `in_stock`, `search`, `page`, `limit` |
| GET | `/api/products/[id]` | — | Full detail: images, description_images, variations, categories |
| POST | `/api/products` | admin | Creates product |
| PUT | `/api/products/[id]` | admin | Updates product fields |
| DELETE | `/api/products/[id]` | admin | Cascades to variations + images |

### Variations

| Method | Route | Auth | Notes |
|---|---|---|---|
| GET | `/api/products/[id]/variations` | — | All variations for a product |
| PUT | `/api/variations/[id]` | admin | Update price, stock, sale flag |
| DELETE | `/api/variations/[id]` | admin | |

### Categories

| Method | Route | Auth | Notes |
|---|---|---|---|
| GET | `/api/categories` | — | Param: `brand_id` |
| POST | `/api/categories` | admin | |
| DELETE | `/api/categories/[id]` | admin | |

### Images

| Method | Route | Auth | Notes |
|---|---|---|---|
| POST | `/api/images/upload` | admin | Multipart: `file`, `type` (product/variation/description), `entity_id`, `sort_order?` |
| DELETE | `/api/images/[id]` | admin | Param: `table` — deletes DB record + Storage file |
| PUT | `/api/images/reorder` | admin | Body: `{ table, updates: { id, sort_order }[] }` |

### Admin

| Method | Route | Auth | Notes |
|---|---|---|---|
| GET | `/api/admin/me` | session | Returns user if admin, else 401/403 |
| POST | `/api/admin/users` | admin | Body: `{ user_id }` |
| DELETE | `/api/admin/users/[userId]` | admin | Cannot remove self |

---

## Auth Flow

```
User submits email + password
  → supabase.auth.signInWithPassword()  (browser, anon key)
  → fetch('/api/admin/me')              (checks admins table server-side)
  → if 200: redirect to /dashboard
  → if 403: sign out, show "no admin access"
```

The `proxy.ts` file runs on every non-API, non-static request and:
- Refreshes the session cookie
- Redirects unauthenticated users away from `/dashboard`, `/products`, `/categories`, `/flagged`
- Redirects authenticated users away from `/login`

---

## Supabase Storage

Bucket: `product-images` (public)

URL pattern:
```
https://zgcekcoatiskqbdruadg.supabase.co/storage/v1/object/public/product-images/{path}
```

Existing paths from migration:
```
clean-products/products/{sku}/{filename}.jpg
clean-products/variations/{sku}/{filename}.jpg
clean-products/descriptions/{sku}/{filename}.jpg
```

New uploads from the dashboard go to:
```
uploads/{type}s/{entity_id}/{timestamp}.{ext}
```

---

## Dashboard Pages

### Login (`/login`)
Client component. Signs in with Supabase Auth, then hits `/api/admin/me` to verify admin access before navigating to dashboard. If admin check fails, signs the user back out.

### Overview (`/dashboard`)
Server component. Runs 6 parallel Supabase count queries + 1 recent products query. Displays stat cards and a recently-added table.

### Products (`/products`)
Server component with a `<Suspense>`-wrapped client filter bar. Filters are URL-driven (search params) so they're bookmarkable and work on refresh. Pagination is 50 per page.

### Product Detail (`/products/[id]`)
Server component for data fetching; `ProductEditForm` is a client component for the edit form. Displays:
- Left column: image grid (sorted by `sort_order`), category pills
- Right column: edit form in a white card
- Bottom: variation preview table with link to full variation editor

### Variations (`/products/[id]/variations`)
Server component table where each row is a `VariationRow` client component. Rows toggle between display and inline-edit mode. Edits are patched to `/api/variations/[id]`.

### Categories (`/categories`)
Server component + `CategoryManager` client component. Inline add form + delete per row.

### Flagged (`/flagged`)
Reads JSON files from `migration/reshaped/` at render time using `fs.readFileSync`. Each file maps to a `FlaggedProductImporter` client component — a collapsible group with per-product import buttons that POST to `/api/products`.

---

## Design System

### Colors
- Page background: `#f7f8fa`
- Sidebar: `#0f1117`
- Cards/panels: `white` with `border-gray-100 shadow-sm`
- Text hierarchy: `gray-900` → `gray-700` → `gray-400`
- Success: `emerald-600` with `emerald-500` dot
- Danger: `red-400` / `red-500`

### Patterns
- **Section labels**: `text-[11px] font-semibold text-gray-400 uppercase tracking-widest`
- **Cards**: `bg-white rounded-xl border border-gray-100 shadow-sm`
- **Inputs**: `h-9 rounded-lg border border-gray-200 focus:ring-2 focus:ring-gray-900/10`
- **Status dots**: `w-1.5 h-1.5 rounded-full` inline with label text — no pill badges
- **Hover reveal**: `group` on row, `opacity-0 group-hover:opacity-100` on action buttons

---

## Gotchas Encountered

### `cp -r` breaks `.bin` symlinks
When the project was scaffolded in a temp directory and copied with `cp -r`, all symlinks in `node_modules/.bin/` were resolved and copied as plain files. The plain files have relative paths that only work from the original symlink target directory (`node_modules/next/dist/bin/`), not from `.bin/`. Fix: delete `node_modules` and run `npm install` fresh.

### Next.js 16 renames `middleware.ts` → `proxy.ts`
In Next.js 16, the middleware file convention changed. The file must be named `proxy.ts` and export a function named `proxy` (or a default export). A file named `middleware.ts` triggers a deprecation warning; the exported function being named `middleware` instead of `proxy` causes a hard error.

### `tsc` bin wrapper was broken after copy
Same symlink issue as above — `node_modules/.bin/tsc` was a plain file with `require('../lib/tsc.js')` which resolved to `node_modules/lib/tsc.js` (doesn't exist) instead of `node_modules/typescript/lib/tsc.js`. Workaround before reinstall: `node node_modules/typescript/lib/tsc.js --noEmit`.

### Supabase `single()` returns `PGRST116` for missing rows
When `.single()` finds no row it returns error code `PGRST116` — not a generic 500. API routes map this to 404:
```ts
const status = error.code === 'PGRST116' ? 404 : 500
```

### `searchParams` in Next.js 16 server components is a Promise
In Next.js 16, `searchParams` passed to page components is `Promise<SearchParams>`, not `SearchParams` directly. Must be awaited:
```ts
export default async function Page({ searchParams }: { params: Promise<SearchParams> }) {
  const sp = await searchParams
}
```
Same applies to `params`.

### Flagged page reads files at render time
`fs.readFileSync` in a server component runs at request time in dev and at build time in production (if statically generated). Since the flagged page reads local JSON files that may be added after deploy, it should not be statically cached. Next.js 16 server components with `fs` reads are dynamic by default, so this works correctly in dev without any extra config.

---

## Data Already in the DB

- **589 products** from bikershades.com
- **5,784 variations** across variable products
- **2,646 images** in Supabase Storage (`product-images` bucket)
- **1 brand**: `bikershades`
- **63 categories**
- **58 flagged products** — not auto-imported, need manual review via `/flagged`

---

## What's Not Built Yet

- **MCP server** — planned at `/api/mcp`. Tools: `list_products`, `get_product`, `update_product`, `list_variations`, `update_variation`, `update_stock`, `list_categories`, `get_catalog_stats`
- **Image upload UI** — the API route exists (`/api/images/upload`) but there's no UI in the dashboard to use it yet
- **Orders / reviews** — schema is catalog-only; no order management or review system
- **Toast notifications** — save/import feedback is inline text only
- **Mobile layout** — sidebar and tables are not responsive
- **Variation price display** — edit mode shows raw cents; should show formatted dollars
