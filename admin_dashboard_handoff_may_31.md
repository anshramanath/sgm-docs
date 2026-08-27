# Admin Dashboard Handoff (May 31, 2026)
_Last updated: 2026-05-31_

Everything you need to build an admin dashboard on top of the seeded Supabase database. Read this in full before writing a line of code — the data has specific shapes, quirks, and constraints that will affect every screen you build.

---

## Environment

```env
SUPABASE_URL=https://<project>.supabase.co
SUPABASE_SERVICE_KEY=<service_role key>   # bypasses RLS — backend only
SUPABASE_ANON_KEY=<anon key>              # for auth flows only
DATABASE_URL=postgresql://postgres:<password>@db.<project>.supabase.co:5432/postgres
```

The frontend **never** hits Supabase directly. All reads and writes go through a Next.js backend using the `service_role` key. The anon key is only used to drive Supabase Auth sign-in flows.

---

## Authentication & Admin Gating

Supabase Auth handles identity. The `admins` table handles authorization.

```sql
create table admins (
  user_id uuid primary key references auth.users(id)
);
```

**How it works:**
1. User signs in via Supabase Auth (email + password)
2. Backend checks `SELECT 1 FROM admins WHERE user_id = auth.uid()`
3. If no row → 403. If row exists → proceed.

**There is no public signup.** The Supabase Auth signup endpoint is publicly accessible by default — do not expose it. Admins must be added manually by inserting a row into the `admins` table after creating the user via the Supabase dashboard or the admin API.

**To add an admin:**
```sql
-- After creating user in Supabase Auth dashboard:
INSERT INTO admins (user_id) VALUES ('<user uuid from auth.users>');
```

**Session pattern:** Issue a short-lived JWT on sign-in, store in httpOnly cookie. On every admin API request, verify the JWT and check the `admins` table. Never trust the client to assert admin status.

---

## Database Schema

### `brands`

```sql
id       uuid  PK
name     text  UNIQUE NOT NULL   -- "BikerShades"
slug     text  UNIQUE NOT NULL   -- "bikershades" (matches Supabase Storage bucket name)
```

Currently 1 row. The slug is the Supabase Storage bucket name — changing it would break all image URLs. Multi-brand support is built in structurally; each brand gets its own bucket.

---

### `categories`

```sql
id        uuid  PK
brand_id  uuid  FK → brands(id)
parent_id uuid  FK → categories(id)  -- NULL = root
name      text  NOT NULL
slug      text  NOT NULL
```

Self-referential tree. `parent_id = NULL` means root level.

**Current seeded state:**
- 54 categories total
- 8 root categories (depth 1)
- 18 at depth 2
- 28 at depth 3
- Max depth: 3

**Root categories (depth 1):**
| Name | Slug |
|------|------|
| Sunglasses | `sunglasses` |
| Bifocals | `bifocals` |
| Prescription | `prescription` |
| Transitions | `transitions` |
| Sale | `sale` |
| Accessories | `accessories` |
| Combo Sets | `combo-sets` |
| Try Before You Buy | `try-before-you-buy` |

**Important:** There is no unique constraint on `(brand_id, slug)` — the same slug can appear under different parents (e.g. `7eye` under both Sunglasses > Brands and Prescription > Rx Brands). Dedup key at import time was `(parent_id, slug)`. When querying or displaying, always use the full path, not just the slug.

**Tree query (recursive):**
```sql
WITH RECURSIVE tree AS (
  SELECT id, name, slug, parent_id, 1 AS depth,
         ARRAY[slug] AS path
  FROM categories
  WHERE parent_id IS NULL AND brand_id = $1

  UNION ALL

  SELECT c.id, c.name, c.slug, c.parent_id, t.depth + 1,
         t.path || c.slug
  FROM categories c
  JOIN tree t ON c.parent_id = t.id
)
SELECT * FROM tree ORDER BY path;
```

---

### `products`

```sql
id                  uuid    PK
brand_id            uuid    FK → brands(id)
name                text    NOT NULL
slug                text    NOT NULL        -- URL routing
sku                 text    UNIQUE NOT NULL
wp_url              text                    -- old WordPress permalink (for 301 redirects)
product_url         text                    -- WooCommerce API URL (for debugging)
description         text                    -- plain text, stripped of HTML
summary             text[]                  -- bullet points parsed from <li> tags
attributes          jsonb                   -- [{name, terms:[{name, slug}]}]
sale                boolean NOT NULL
regular_price_cents int     NOT NULL        -- price in cents, e.g. 1999 = $19.99
sale_price_cents    int     NOT NULL
average_rating      float   NOT NULL
review_count        int     NOT NULL
in_stock            boolean NOT NULL
weight              float                   -- in oz (weight_unit)
weight_unit         text    DEFAULT 'oz'
length              float                   -- in inches (dimension_unit)
width               float
height              float
dimension_unit      text    DEFAULT 'in'
```

**581 products seeded.** 489 variable (have variations), 92 simple (no variations).

**Price handling:** Always store and retrieve in cents. Display as `$${(price_cents / 100).toFixed(2)}`. Never store floats for money.

**`attributes` jsonb shape:**
```json
[
  {
    "name": "color",
    "terms": [
      { "name": "Blue Green Smoke", "slug": "blue-green-smoke" },
      { "name": "Black Clear", "slug": "black-clear" }
    ]
  },
  {
    "name": "power",
    "terms": [
      { "name": "1.00", "slug": "1.00" },
      { "name": "2.50", "slug": "2.50" }
    ]
  }
]
```

9 attribute types exist: `color`, `power`, `transition-type`, `frame-color`, `size`, `brand`, `foam-type`, `quantity-packs-3`, `lens-genuine-transition`. `color` is the dominant one (459/581 products). `attributes` can be `null` for simple products with no variants.

**`summary` array:** Parsed from WooCommerce `short_description` `<li>` tags. 187/581 products have an empty summary — this is expected. Frontend should hide the bullet list when empty.

**`wp_url`:** The old bikershades.com product URL. Use this to build 301 redirects from old Google-indexed URLs. Format: `https://bikershades.com/shop/...`.

---

### `variations`

```sql
id                  uuid    PK
product_id          uuid    FK → products(id)
slug                text    NOT NULL
sku                 text    UNIQUE NOT NULL
variation           text[]                  -- e.g. ["black-clear", "1.00"]
wp_url              text
product_url         text
description         text
sale                boolean NOT NULL
regular_price_cents int     NOT NULL
sale_price_cents    int     NOT NULL
in_stock            boolean NOT NULL
weight              float
weight_unit         text    DEFAULT 'oz'
length              float
width               float
height              float
dimension_unit      text    DEFAULT 'in'
image_src           text                    -- Supabase storage URL (nullable)
image_name          text
```

**5,649 variations seeded.** Each variation represents one specific color/power/size combo of a parent product.

**`variation` text[]:** Flat array of attribute term slugs in the same order as the parent's `attributes` array. Example: if a product has `[color, power]` attributes, then `["black-clear", "1.00"]` means black-clear lens + 1.00 diopter.

**Image:** Variations have at most 1 image stored inline on the row (`image_src`, `image_name`). `image_src` is the full Supabase public URL or `null`. 108 variations have `image_src = null` — the frontend falls back to the parent product's first `product_images` row.

**Prices:** Variation prices can differ from the parent. Always use variation price when displaying a selected color/power. Fall back to parent price only for display on the listing page before a variation is selected.

**In stock:** `in_stock` is per-variation, not just per-product. A product can be in stock overall but have specific variations out of stock.

---

### `product_images`

```sql
id          uuid  PK
product_id  uuid  FK → products(id)
src         text  NOT NULL   -- full Supabase public URL
name        text  NOT NULL   -- image filename, e.g. "Backspin-Front.jpg"
sort_order  int   NOT NULL   -- 0-indexed, ascending
```

**2,618 rows.** Multiple images per product, ordered by `sort_order`. The first image (`sort_order = 0`) is the primary/hero image. Always fetch ordered: `ORDER BY sort_order ASC`.

Note: `src` values are Supabase public URLs pointing to the `bikershades` bucket. They're stable — don't cache-bust them unless you're changing the actual file.

---

### `description_images`

```sql
id           uuid  PK
product_id   uuid  FK → products(id)  -- nullable
variation_id uuid  FK → variations(id) -- nullable
src          text  NOT NULL
name         text  NOT NULL
sort_order   int   NOT NULL
-- CHECK: exactly one of product_id or variation_id is non-null
```

**935 rows.** Images extracted from product/variation description HTML in WooCommerce. These were embedded inline in the description body — they're now stored separately so the description field is clean plain text.

When rendering a product description, fetch these and inject them back into the display wherever appropriate (or render them as a separate image block below the description text).

The check constraint enforces that every row belongs to exactly one of a product or variation — never both, never neither.

---

### `product_categories`

```sql
product_id   uuid  FK → products(id)     -- composite PK
category_id  uuid  FK → categories(id)  -- composite PK
```

**1,082 rows.** Many-to-many junction. A product can belong to multiple categories, and a category can contain many products.

**Important:** Only leaf-node categories are assigned. No product is assigned to a branch node. This means you can always build breadcrumb navigation by walking up the tree via `parent_id` — you'll never hit a product assignment in the middle.

**Products per category** range from 1 to ~100+. The `sunglasses` subtree has the most. See `docs/shaped-items/categories.md` for full counts from the WooCommerce source.

---

## Data Relationships (mental model)

```
brands
  └── categories (tree via parent_id)
  └── products
        ├── product_categories → categories (leaf nodes only)
        ├── product_images (ordered gallery)
        ├── description_images (from description HTML)
        └── variations
              ├── image_src / image_name (inline, 1 max)
              └── description_images (rare — only 4 variations had these)
```

---

## Admin Dashboard — Feature Spec

### 1. Products List

The main table. Should be the landing page of the dashboard.

**Columns to show:**
- Hero image (first `product_images` row, `sort_order = 0`)
- Name
- SKU
- Price (regular / sale)
- Sale badge (`sale = true`)
- Stock status (`in_stock`)
- Variation count
- Category paths (abbreviated)

**Filtering:**
- By category (tree picker — show full path)
- By sale status
- By stock status
- By attribute (e.g. filter to all "bifocal" products via `power` attribute)
- Text search on name / SKU

**Sorting:** name, price, rating, review count, variation count

**Query pattern:**
```sql
SELECT
  p.*,
  pi.src AS hero_image,
  COUNT(DISTINCT v.id) AS variation_count,
  COUNT(DISTINCT pc.category_id) AS category_count
FROM products p
LEFT JOIN product_images pi ON pi.product_id = p.id AND pi.sort_order = 0
LEFT JOIN variations v ON v.product_id = p.id
LEFT JOIN product_categories pc ON pc.product_id = p.id
WHERE p.brand_id = $1
GROUP BY p.id, pi.src
ORDER BY p.name ASC;
```

---

### 2. Product Detail / Edit

Full product view with all associated data.

**Sections:**

**Core fields** (editable inline):
- Name, slug, SKU
- Description (rich text editor — plain text in DB, but admin can format for display)
- Summary (bullet list editor — maps to `text[]`)
- Regular price, sale price (show as dollars, store as cents)
- Sale toggle
- In stock toggle
- Weight (with unit)
- Dimensions L × W × H (with unit)

**Images:**
- Drag-to-reorder gallery (`sort_order` is the position)
- Add image (upload to Supabase Storage → insert row)
- Delete image (delete row → optionally delete from storage)
- Hero image is always `sort_order = 0`

**Description images:**
- Separate section, listed in order
- Readonly for now — these came from WooCommerce HTML

**Categories:**
- Multi-select tree picker
- Shows current category assignments as full paths (e.g. `Sunglasses > Brands > ProSport`)
- Add / remove category assignments

**Attributes:**
- Display current `attributes` jsonb as a readable table
- Each attribute row: name + list of terms
- For now: readonly (modifying attributes requires re-indexing variations)

**Variations table:**
- List all variations with: SKU, `variation[]` slugs, price, stock, image thumbnail
- Click to expand/edit a single variation

**Debug links:**
- `wp_url` → link to original WooCommerce page (useful for owner reference)
- `product_url` → link to WooCommerce API response (useful for data debugging)

---

### 3. Variation Detail / Edit

Accessed from inside a product page.

**Editable fields:**
- Regular price, sale price
- In stock toggle
- Sale toggle
- Weight, dimensions
- Image (upload new or remove)
- Description (plain text)

**Readonly:**
- SKU (changing breaks inventory references)
- `variation[]` slugs (changing breaks attribute-to-variation mapping)
- `slug` (URL routing)

**Image handling:** `image_src` is a Supabase public URL. To replace: upload new file to storage, update `image_src` and `image_name`. To remove: set both to `null` — frontend will fall back to product hero image.

---

### 4. Category Manager

Tree view of all 54 categories.

**Display:** Collapsible tree. Each node shows: name, slug, product count.

**Actions per node:**
- Rename (update `name`)
- Edit slug (update `slug` — warn that this breaks URLs if the site is live)
- Move (change `parent_id`)
- Add child category (insert with this node's `id` as `parent_id`)
- Delete (only if no products assigned and no children)

**Product count per category:**
```sql
SELECT c.id, c.name, COUNT(pc.product_id) AS product_count
FROM categories c
LEFT JOIN product_categories pc ON pc.category_id = c.id
GROUP BY c.id, c.name;
```

**Important rule:** Products are only assigned to leaf nodes. If the admin creates a new child under an existing leaf, they must move the products from the old leaf to the new child — the dashboard should warn about this.

---

### 5. Image Manager

Global view of all images in Supabase Storage.

**Tabs:**
- Product images — grouped by product
- Description images — grouped by product/variation
- Variation images — inline on variation rows

**Per image actions:**
- Preview
- Replace (upload new → update `src`)
- Delete (delete from storage + delete DB row)
- Reorder (drag within a product's gallery)

**Storage:** All images live flat in the `bikershades` bucket. Filename format: `2024_05_image-name.jpg` (date-prefixed path from original URL). When uploading new images from the admin, use a consistent naming scheme — recommend `admin_<sku>_<timestamp>.<ext>`.

---

### 6. Stats / Overview Page

Dashboard home or sidebar widget.

**Useful numbers:**
```sql
-- Counts
SELECT
  (SELECT COUNT(*) FROM products) AS products,
  (SELECT COUNT(*) FROM variations) AS variations,
  (SELECT COUNT(*) FROM categories) AS categories,
  (SELECT COUNT(*) FROM products WHERE sale = true) AS on_sale,
  (SELECT COUNT(*) FROM products WHERE in_stock = false) AS out_of_stock,
  (SELECT COUNT(*) FROM variations WHERE in_stock = false) AS variations_out_of_stock,
  (SELECT COUNT(*) FROM product_images) AS product_images,
  (SELECT COUNT(*) FROM variations WHERE image_src IS NULL) AS variations_no_image;
```

**Products by category (top level):**
```sql
SELECT c.name, COUNT(DISTINCT pc.product_id) AS products
FROM categories c
JOIN product_categories pc ON pc.category_id = c.id
WHERE c.parent_id IS NULL
GROUP BY c.id, c.name
ORDER BY products DESC;
```

---

### 7. Admin User Manager

Simple table of who has admin access.

```sql
SELECT a.user_id, u.email, u.created_at
FROM admins a
JOIN auth.users u ON u.id = a.user_id;
```

**Actions:**
- Add admin (create Supabase Auth user via admin API → insert into `admins`)
- Remove admin (delete from `admins` — does NOT delete the auth user)

Note: only a super-admin (you, via service_role key) should be able to manage this table. Don't expose this to regular admins unless intentional.

---

## Data Quirks to Know

### Prices are always in cents
`regular_price_cents = 1999` means $19.99. Never store or display raw. Always divide by 100 for display, multiply by 100 for storage. Both product and variation rows have separate price fields — use variation price when a variant is selected.

### `sale` flag logic
A product or variation has `sale = true` only if `on_sale == true AND sale_price < regular_price`. `sale_price_cents` always has a value even when `sale = false` — it may be a staged future sale price. Only use `sale_price_cents` for display when `sale = true`.

### Simple vs variable products
A product is variable if it has variations. Check `COUNT(variations)` — don't rely on any type field (there isn't one). Simple products have `attributes = null` or `attributes = []`.

### 108 variations with no image
`image_src IS NULL` on 108 variation rows. This is expected — some variation images 404'd from the WooCommerce server, others never had variation-specific images. Frontend falls back to `product_images` where `sort_order = 0`. The admin should surface this clearly rather than showing a broken image placeholder.

### `summary` can be empty
187/581 products have `summary = null` or `summary = {}`. Frontend hides the bullet list in this case. Admin editor should allow adding summary bullets even when none exist.

### `description` can be empty
17/581 products have no description text. Admin should handle null gracefully in the editor.

### `wp_url` is a 301 redirect source
These are old Google-indexed URLs. Do not delete them from the DB — they're needed to build the redirect map when the new site goes live.

### Category slugs are not globally unique
The same slug (e.g. `7eye`) can appear under different parents. Always identify categories by their `id` or full path, never by slug alone.

### `attributes` jsonb has known data quality issues
Some attribute slugs are not URL-safe:
- `frame-color`: `"Matte Black"` has slug `Matte Black` (space, mixed case) — should be `matte-black`
- `quantity-packs-3`: slugs like `1 pack (3 masks total)` have spaces and parentheses
- `lens-genuine-transition`: slugs are internal WooCommerce labels that don't match display names

These are carried over from WooCommerce as-is. If the frontend uses attribute slugs for URL routing (e.g. `/sunglasses?color=black-clear`), these edge cases will need normalization before the site goes live.

### `color` attribute has capitalization duplicates
`"Black"` appears as both `black` and `Black`, `"Tortoise"` as both `tortoise` and `Tortoise`. These are distinct DB entries but represent the same physical color. The admin may want to merge them. Don't silently deduplicate — ask the owner which slug is canonical.

---

## Image URLs

All images are public Supabase Storage URLs. Format:
```
https://<project>.supabase.co/storage/v1/object/public/bikershades/<filename>
```

Example: `https://<project>.supabase.co/storage/v1/object/public/bikershades/2024_05_Hangar-Gunmetal-Front.jpg`

These URLs are stable — no expiry, no signing required (bucket is public). If you need image transforms (resize, crop, format), use Supabase's image transform API:
```
https://<project>.supabase.co/storage/v1/render/image/public/bikershades/<filename>?width=400&quality=80
```

---

## API Design Recommendations

Build a thin Next.js API layer in front of Supabase. Never expose `service_role` key to the browser.

**Suggested routes:**

```
GET    /api/admin/products          list with filters + pagination
GET    /api/admin/products/:id      full product with variations + images + categories
PATCH  /api/admin/products/:id      update product fields
GET    /api/admin/products/:id/variations
PATCH  /api/admin/variations/:id    update variation fields
GET    /api/admin/categories        full tree
POST   /api/admin/categories        create category
PATCH  /api/admin/categories/:id    rename / move
DELETE /api/admin/categories/:id    delete (if empty)
GET    /api/admin/stats             dashboard overview counts
GET    /api/admin/admins            list admin users
POST   /api/admin/admins            add admin
DELETE /api/admin/admins/:id        remove admin
```

All routes should:
1. Verify session JWT
2. Check `admins` table
3. Use `service_role` client for DB operations
4. Return consistent error shapes

---

## Suggested Tech Stack

- **Framework:** Next.js App Router (backend API routes + admin frontend in one repo)
- **Auth:** Supabase Auth (already configured)
- **DB client:** `@supabase/supabase-js` with `service_role` key on server
- **UI:** shadcn/ui — good table, dialog, and form primitives; works well for data-heavy admin UIs
- **Rich text:** Tiptap or Lexical for description editing
- **Image upload:** Supabase Storage JS client (`supabase.storage.from('bikershades').upload(...)`)
- **Tree UI:** Build a custom recursive component — category trees are shallow enough (max depth 3) that a custom component is cleaner than a library

---

## Launch Checklist

Before exposing the admin to the owner:

- [ ] Add owner as admin via `admins` table
- [ ] Test sign-in flow end to end
- [ ] Verify all images render (check a sample of `product_images.src` URLs in browser)
- [ ] Test product edit → save → verify DB updated correctly
- [ ] Test variation stock toggle
- [ ] Confirm category tree renders correctly with product counts
- [ ] Build 301 redirect map from `wp_url` values before go-live
- [ ] Confirm `sale` badge logic: `sale = true AND sale_price < regular_price`
- [ ] Resolve `frame-color` slug issue (`Matte Black`) before attribute-based URL routing goes live
