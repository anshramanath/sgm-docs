# Admin Dashboard — Handoff (May 30, 2026)

Everything needed to build the backend admin dashboard server.

---

## What This App Is

A Next.js app that serves three roles:
1. **Admin dashboard** — web UI for managing the product catalog
2. **API server** — all 3 brand frontends call this instead of hitting Supabase directly
3. **MCP server** — exposes tools for AI agent access to the catalog

---

## Architecture

```
[bikershades frontend]  ──┐
[brand2 frontend]       ──┤──→  [this Next.js app]  ──→  [Supabase DB + Storage]
[brand3 frontend]       ──┘      (Admin + API + MCP)
```

- Frontends never touch Supabase directly — always go through this app
- This app uses `service_role` key — bypasses RLS entirely
- RLS is enabled on all tables as a safety net against accidental direct access
- Supabase Auth handles login — `admins` table is the allowlist

---

## Environment Variables

```env
SUPABASE_URL=https://zgcekcoatiskqbdruadg.supabase.co
SUPABASE_SERVICE_ROLE_KEY=<service_role key — NOT the anon key>
NEXT_PUBLIC_SUPABASE_URL=https://zgcekcoatiskqbdruadg.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=<anon key — only for client-side auth UI>
```

`service_role` lives server-side only. Never expose it to the browser. `anon` key is only needed if you use Supabase's client-side auth helpers for the login form.

---

## Database Schema

Full SQL: `migration/db/001_initial_schema.sql`

### Tables

| Table | Key Fields |
|---|---|
| `brands` | `id`, `name` |
| `categories` | `id`, `brand_id`, `name` |
| `products` | `id`, `brand_id`, `name`, `slug`, `sku`, `wp_url`, `product_url`, `description`, `summary text[]`, `attributes jsonb`, `sale`, `regular_price_cents`, `sale_price_cents`, `average_rating`, `review_count`, `in_stock`, `weight`, `weight_unit`, `length`, `width`, `height`, `dimension_unit` |
| `product_categories` | `product_id`, `category_id` (junction) |
| `variations` | `id`, `product_id`, `slug`, `sku`, `variation text[]`, `wp_url`, `product_url`, `description`, `sale`, `regular_price_cents`, `sale_price_cents`, `in_stock`, `weight`, `weight_unit`, `length`, `width`, `height`, `dimension_unit`, `image_src`, `image_name` |
| `product_images` | `id`, `product_id`, `src`, `name`, `sort_order` |
| `description_images` | `id`, `product_id?`, `variation_id?`, `src`, `name`, `sort_order` |
| `admins` | `user_id → auth.users(id)` |

### Key field notes

**`attributes` (jsonb on products)**
Grouped by type — enables listing-page color swatch rendering without joining variations:
```json
[
  { "name": "color", "terms": ["black-clear", "blue-smoke"] },
  { "name": "size",  "terms": ["small-medium", "large-xlarge"] }
]
```
Query all products with a blue option:
```sql
SELECT * FROM products
WHERE attributes @> '[{"name": "color", "terms": ["blue-smoke"]}]';
```

**`variation text[]`**
The specific option combo for this variation: `["black-clear", "1-00"]`.

**`image_src` / `image_name` on variations**
Variations have at most 1 image — stored inline on the row, not a separate table. `null` if no image — frontend falls back to parent product images.

**`description_images`**
Images extracted from product/variation description HTML. Either `product_id` OR `variation_id` is set — never both. CHECK constraint enforces this.

**`summary text[]`**
Bullet points parsed from product short description. Render as a feature list, hide section if empty.

---

## Auth Flow

No public signup. Admins are added manually.

**Login:**
1. User submits email + password
2. Authenticate via Supabase Auth: `supabase.auth.signInWithPassword()`
3. Check if `user_id` exists in `admins` table
4. If yes → grant access. If no → reject (even if auth succeeded)

```ts
const { data: { user }, error } = await supabase.auth.signInWithPassword({ email, password })
if (error || !user) return unauthorized()

const { data: admin } = await supabase
  .from('admins')
  .select('user_id')
  .eq('user_id', user.id)
  .single()

if (!admin) return forbidden()
```

**Adding an admin:**
1. Supabase dashboard → Authentication → Users → Add user
2. SQL editor: `INSERT INTO admins (user_id) VALUES ('<user-id>');`

---

## Supabase Client Setup

Two clients — server (service_role) and browser (anon):

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

## Image Storage

- **Bucket:** `bikershades` (public)
- **Structure:** flat — all images sit directly in the bucket root
- **URL format:** `https://zgcekcoatiskqbdruadg.supabase.co/storage/v1/object/public/bikershades/{filename}`
- When brand2 is added → create a `brand2` bucket, same pattern

---

## Data In The DB (post-migration)

| | Count |
|---|---|
| Brand | 1 (bikershades) |
| Categories | 63 |
| Clean products | 586 |
| Variations | ~5,750 |
| Images in storage | ~2,559 |
| Flagged (pending owner review) | 61 |

---

## API Routes

All server-side. Never use `service_role` in a client component.

### Products
```
GET    /api/products                list with pagination + filters
GET    /api/products/[id]           single product with variations + images
POST   /api/products                create
PUT    /api/products/[id]           update
DELETE /api/products/[id]           delete (cascades)
```

**Full product with everything:**
```ts
const { data } = await supabaseAdmin
  .from('products')
  .select(`
    *,
    product_images(src, name, sort_order),
    description_images(src, name, sort_order),
    variations(
      *,
      description_images(src, name, sort_order)
    ),
    product_categories(categories(id, name))
  `)
  .eq('id', productId)
  .single()
```

### Variations
```
GET    /api/products/[id]/variations
PUT    /api/variations/[id]
DELETE /api/variations/[id]
```

### Categories
```
GET    /api/categories
POST   /api/categories
DELETE /api/categories/[id]
```

### Images
```
POST   /api/images/upload       upload to Supabase Storage, return URL
DELETE /api/images/[id]         delete record + remove from storage
PUT    /api/images/reorder       update sort_order for a set of images
```

### Admin
```
GET    /api/admin/me            check if current session is admin
POST   /api/admin/users         add to admins table
DELETE /api/admin/users/[id]    remove admin access
```

---

## MCP Server

Exposes catalog tools for AI agent access. Mount at `/api/mcp`.

**Suggested tools:**
```
list_products(brand_id?, category?, in_stock?, page?)
get_product(id or sku)
update_product(id, fields{})
list_variations(product_id)
update_variation(id, fields{})
update_stock(sku, in_stock)
list_categories(brand_id?)
get_catalog_stats()
```

```ts
import { Server } from '@modelcontextprotocol/sdk/server/index.js'
```

---

## Admin Dashboard Pages

| Route | Purpose |
|---|---|
| `/login` | Email + password |
| `/dashboard` | Overview — counts, flagged items |
| `/products` | Paginated list with search + filter |
| `/products/[id]` | Edit product, manage images, view variations |
| `/categories` | Manage categories |
| `/flagged` | Review the 61 products not yet imported |

---

## Flagged Products (Not Yet In DB)

| File | Count | Issue |
|---|---|---|
| `invalid_sku_products_shape.json` | 22 | SKU null or has a space |
| `no_weight_products_shape.json` | 14 | Missing or zero weight |
| `zero_price_products_shape.json` | 8 | Price is zero |
| `missing_attributes_products_shape.json` | 7 | Variation option data missing |
| `duplicate_name_products_shape.json` | 6 | Same name as another product |
| `no_image_products_shape.json` | 3 | No images |
| `multiple_variation_images_products_shape.json` | 1 | Variation has >1 image |

All files in `migration/reshaped/`. Owner review questions in `migration/reshaped-problems/treatment.md`.

---

## Tech Stack Recommendation

| Layer | Choice |
|---|---|
| Framework | Next.js 14+ (App Router) |
| Language | TypeScript |
| DB client | `@supabase/supabase-js` with service_role |
| Auth | Supabase Auth + admins table |
| MCP | `@modelcontextprotocol/sdk` |

---

## Known Limitations

- `wp_url` on products/variations — old bikershades.com URLs, keep for 301 redirects
- `average_rating` / `review_count` — seeded from WooCommerce, not a real review system
- `sale` flag vs sale page — the old sale page was manually curated, not auto-generated from the flag. Build your own filter logic.
- No orders, reviews, or user accounts yet — catalog only
