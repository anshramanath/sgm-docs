# Backend Handoff — Admin Dashboard + API + MCP Server (May 29, 2026)

This document covers everything needed to build the Next.js backend app. It serves three roles:
1. **Admin dashboard** — web UI for managing the product catalog
2. **API server** — backend for all 3 brand frontends (they never hit Supabase directly)
3. **MCP server** — exposes tools for AI agent access to the catalog

---

## Architecture

```
[bikershades frontend]  ──┐
[brand2 frontend]       ──┤──→  [this Next.js app]  ──→  [Supabase DB]
[brand3 frontend]       ──┘         (API + Admin)         (Postgres + Auth + Storage)
                                    (MCP server)
```

- **Frontends** call this app's API routes — they never use Supabase keys
- **This app** uses the `service_role` key — bypasses RLS entirely
- **RLS is enabled** on all tables as a safety net (blocks any accidental direct client access)
- **Supabase Auth** handles login — `admins` table gates who has admin access
- **Supabase Storage** hosts all product images (bucket: `product-images`, public)

---

## Tech Stack

| Layer | Choice |
|---|---|
| Framework | Next.js 14+ (App Router) |
| Language | TypeScript |
| DB client | `@supabase/supabase-js` with `service_role` key |
| Auth | Supabase Auth (email/password) + `admins` table check |
| Session | NextAuth.js or Supabase Auth session (your choice) |
| MCP | `@modelcontextprotocol/sdk` |
| Styling | Tailwind CSS (or whatever you prefer for the dashboard) |

---

## Environment Variables

```env
SUPABASE_URL=https://zgcekcoatiskqbdruadg.supabase.co
SUPABASE_SERVICE_ROLE_KEY=<service_role key — NOT the anon key>
NEXT_PUBLIC_SUPABASE_URL=https://zgcekcoatiskqbdruadg.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=<anon key — only for client-side auth UI>
```

> The `service_role` key bypasses RLS. Never expose it to the browser. It only lives in server-side code (API routes, server components, MCP handlers).
>
> The `anon` key is only needed if you use Supabase's client-side auth helpers for the login form. If you handle auth entirely server-side, you don't need it at all.

---

## Database Schema

Source of truth: `migration/db/001_initial_schema.sql`

```sql
brands          — id, name
categories      — id, brand_id, name
products        — id, brand_id, name, slug, sku, wp_url, product_url,
                   description, summary text[], attributes jsonb,
                   sale, regular_price_cents, sale_price_cents,
                   average_rating, review_count, in_stock,
                   weight, weight_unit, length, width, height, dimension_unit
product_categories — product_id, category_id  (junction)
variations      — id, product_id, slug, sku, variation text[],
                   wp_url, product_url, description,
                   sale, regular_price_cents, sale_price_cents,
                   in_stock, weight, weight_unit, length, width, height, dimension_unit
product_images  — id, product_id, src, name, sort_order
variation_images — id, variation_id, src, name, sort_order
description_images — id, product_id?, variation_id?, src, name, sort_order
                     CHECK: exactly one of product_id or variation_id is set
admins          — user_id → auth.users(id)
```

### Key field notes

**`attributes` (jsonb on products)**
Grouped by attribute type for listing-page queries without joining variations:
```json
[
  { "name": "color", "terms": ["black-clear", "blue-smoke", "red-clear"] },
  { "name": "size",  "terms": ["small-medium", "large-xlarge"] }
]
```
Query example — find all products that have a blue color option:
```sql
SELECT * FROM products
WHERE attributes @> '[{"name": "color", "terms": ["blue-smoke"]}]';
```

**`variation text[]` (on variations)**
The specific option combo for this variation: `["black-clear", "1-00"]`. Maps positionally to the parent product's `attributes[].terms`.

**`summary text[]`**
Parsed bullet points from the product's short description. Rendered as a feature list on the product page.

**`categories`**
63 unique category names from bikershades.com. Used for top-level nav grouping only. Brand/lens-type/head-size filtering is driven by `attributes` on products, not categories.

**`description_images`**
Images extracted from product/variation description HTML. These are informational/diagram images embedded in the description body. Stored separately so the frontend can render them in the right context.

---

## Auth Flow

No public signup. Admins are added manually to `auth.users` via the Supabase dashboard, then their `user_id` is inserted into the `admins` table.

**Login flow:**
1. User submits email + password
2. Authenticate via Supabase Auth: `supabase.auth.signInWithPassword()`
3. Check if `user_id` exists in `admins` table
4. If yes → grant access. If no → reject (even if auth succeeded)

```ts
// Server-side auth check
const { data: { user }, error } = await supabase.auth.signInWithPassword({
  email, password
})
if (error || !user) return unauthorized()

const { data: admin } = await supabase
  .from('admins')
  .select('user_id')
  .eq('user_id', user.id)
  .single()

if (!admin) return forbidden() // authenticated but not an admin
```

**Adding an admin:**
1. Go to Supabase dashboard → Authentication → Users → Invite user
2. Run in SQL editor: `INSERT INTO admins (user_id) VALUES ('<user-id-from-auth>');`

---

## Data Already in the DB (post-migration)

- **589 clean products** from bikershades.com
- **~6,000 variations** across 486 variable products
- **~2,785 images** in Supabase Storage (`product-images` bucket)
- **1 brand**: `bikershades`
- **63 categories**

The remaining 58 flagged products (invalid SKU, no weight, duplicates, missing attributes, zero price, no image) need manual review and will be imported separately.

---

## API Routes

All routes are server-side only. Never use the service_role key in a client component.

### Products

```
GET    /api/products                    List products (with pagination, filtering)
GET    /api/products/[id]               Single product with variations + images
POST   /api/products                    Create product
PUT    /api/products/[id]               Update product
DELETE /api/products/[id]               Delete product (cascades to variations, images)
```

**Useful query patterns:**

```ts
// List products for a brand with category filter
const { data } = await supabase
  .from('products')
  .select(`
    *,
    product_images(src, name, sort_order),
    product_categories(category_id)
  `)
  .eq('brand_id', brandId)
  .order('name')

// Full product detail with everything
const { data } = await supabase
  .from('products')
  .select(`
    *,
    product_images(src, name, sort_order),
    description_images(src, name, sort_order),
    variations(
      *,
      variation_images(src, name, sort_order),
      description_images(src, name, sort_order)
    ),
    product_categories(
      categories(id, name)
    )
  `)
  .eq('id', productId)
  .single()
```

### Variations

```
GET    /api/products/[id]/variations    List variations for a product
PUT    /api/variations/[id]             Update variation (price, stock, etc.)
DELETE /api/variations/[id]             Delete variation
```

### Categories

```
GET    /api/categories                  List categories for a brand
POST   /api/categories                  Create category
DELETE /api/categories/[id]             Delete category
```

### Images

```
POST   /api/images/upload               Upload image to Supabase Storage, return URL
DELETE /api/images/[id]                 Remove image record + delete from storage
PUT    /api/images/reorder              Update sort_order for a set of images
```

### Admin

```
GET    /api/admin/me                    Check if current session is admin
POST   /api/admin/users                 Add a user to admins table
DELETE /api/admin/users/[userId]        Remove admin access
```

---

## MCP Server

The MCP server exposes catalog tools that AI agents (e.g. Claude Code) can call to read and manage product data. Mount it as a route handler in Next.js.

**Suggested tools to expose:**

```
list_products(brand_id?, category?, in_stock?, page?)
get_product(id or sku)
update_product(id, fields{})
list_variations(product_id)
update_variation(id, fields{})
update_stock(sku, in_stock)
list_categories(brand_id?)
get_catalog_stats()          — counts per table, flagged products, etc.
```

**Setup:**
```ts
import { Server } from '@modelcontextprotocol/sdk/server/index.js'
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js'
```

Mount at `/api/mcp` as a Next.js route handler, or run as a standalone process depending on how your Claude config connects to it.

---

## Supabase Client Setup

Create two clients — one for server use (service_role), one for client auth only.

```ts
// lib/supabase/server.ts — server only, never import in client components
import { createClient } from '@supabase/supabase-js'

export const supabaseAdmin = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
)
```

```ts
// lib/supabase/client.ts — browser only, for auth UI
import { createBrowserClient } from '@supabase/ssr'

export const supabase = createBrowserClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
)
```

---

## Admin Dashboard Pages

Minimum viable pages:

| Route | Purpose |
|---|---|
| `/login` | Email + password login form |
| `/dashboard` | Overview — product counts, flagged items, recent changes |
| `/products` | Paginated product list with search + filter |
| `/products/[id]` | Product detail — edit fields, manage images, view variations |
| `/products/[id]/variations` | Variation list — edit price, stock, images per variation |
| `/categories` | List + create/delete categories |
| `/flagged` | Review the 58 products that weren't auto-imported |

---

## Image URL Pattern

All images are in Supabase Storage, bucket `product-images`. Public URL format:

```
https://zgcekcoatiskqbdruadg.supabase.co/storage/v1/object/public/product-images/{path}
```

Path mirrors the original local structure:
```
clean-products/products/{sku}/{filename}.jpg
clean-products/variations/{sku}/{filename}.jpg
clean-products/descriptions/{sku}/{filename}.jpg
```

---

## Flagged Products (Need Manual Review)

These 58 products were not auto-imported and need review before adding to the DB:

| File | Count | Issue |
|---|---|---|
| `invalid_sku_products_shape.json` | 22 | SKU is null or contains a space |
| `no_weight_products_shape.json` | 12 | Missing weight (needed for shipping calc) |
| `zero_price_products_shape.json` | 8 | Price is zero — likely internal/service products |
| `missing_attributes_products_shape.json` | 7 | Variations with no attribute data |
| `duplicate_name_products_shape.json` | 6 | Same name as another product |
| `no_image_products_shape.json` | 3 | No images at all |

All reshaped JSON files are in `migration/reshaped/`. The admin dashboard's `/flagged` page should load these, let you edit/fix them inline, and import them individually.

---

## Known Limitations / Things to Keep in Mind

- **No orders/reviews yet** — the DB schema is catalog-only. Orders, reviews, and user accounts are future work.
- **`wp_url` on products/variations** — keep these. They're the old bikershades.com URLs used for 301 redirects from Google-indexed pages.
- **`sale` flag vs sale page** — `sale: true` on a product/variation means it has a lower sale price. The bikershades sale page was manually curated and doesn't match the `on_sale` flag 1:1. Build the sale page view with your own filter logic.
- **`average_rating` / `review_count`** — seeded from WooCommerce data. Will need to be connected to a real review system eventually.
- **Description images are shared** — the same image can be referenced in multiple products' descriptions. They're stored once in `description_images` per product/variation that uses them.
