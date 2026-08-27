# Bikershades Migration — Learnings & Build Summary (May 29, 2026)

## What We Built

A full data migration pipeline from bikershades.com (WordPress/WooCommerce) into a clean, normalized schema ready for a new multi-brand app backed by Supabase.

### Pipeline

```
WooCommerce Store API
        ↓
fetch_products.py       → products.json (647 products)
        ↓
fetch_variations.py     → variations.json (6,064 variations)
        ↓
reshape_products.py     → 7 reshaped JSON files (bucketed by issue type)
        ↓
download_images.py      → 2,785 images downloaded locally
        ↓
[next] upload to storage + insert into Supabase
```

---

## WooCommerce Store API

- **Endpoint:** `https://bikershades.com/wp-json/wc/store/products`
- **Pagination:** `?per_page=100&page=N` — stop when response is empty list
- **Variations not returned via pagination** — WooCommerce only returns parent products. Variations must be fetched individually via `/products/{variation_id}`
- **Prices are strings in cents** — `"1999"` = $19.99, always USD
- **`stock_status` is always null** — actual stock is in `is_in_stock` (bool)
- **`on_sale` alone is not enough** — must also check `sale_price < regular_price`. Owners can stage a sale price without activating it
- **Sale page is manually curated** — not an automatic filter of all `on_sale` products. ~81% of products have `on_sale: true` but the sale page shows far fewer
- **Variation stub in parent product** — each variation listed under a parent has `{ id, attributes: [{name, value}] }`. The `value` is the slug for that attribute option (e.g. `"black-clear"`). This is the source of truth for attribute combos, not the variation's own `variation` string (which is sometimes empty)

---

## Data Shape Learnings

### Products
- 647 total: 489 variable, 157 simple, 1 prescription
- Variable products have `attributes[].terms[]` driving color swatches on the frontend
- `has_variations: true` on 9 simple products — WooCommerce misconfiguration, ignore it
- 38 variations had empty `variation` strings — resolved by falling back to stub attribute values from parent

### Variations
- 6,064 variations across 486 variable parents
- Each variation can have its own images, price, stock, SKU, and weight
- `summary`, `averageRating`, `reviewCount` always empty/zero at variation level — dropped
- Variations are cartesian product of all attribute options (e.g. 4 colors × 11 transition types = 44 variations)

### Images
- Products have 0–18 images each, most have 2–7
- 3 products have no images
- 56 variations have no images — frontend falls back to parent product images
- Description HTML contains embedded images — extracted separately into `descriptionImgs[]`
- 990 total description image references across 587 products, but only 40 unique URLs — same diagrams reused across many products
- 86 variation images + 10 description images returned 404 — deleted from server ~2018, not recoverable

### Categories
- WooCommerce uses categories as a catch-all: brand, lens type, head size, product type, navigation, etc.
- 63 unique category pages exist on the live site
- In the new schema: categories only used for top-level nav grouping. Brand filtering via `brands` table, lens/head-size filtering via `attributes[]`

---

## Reshaping Decisions

### Kept
| Field | Notes |
|---|---|
| `name` | HTML entity decoded, tags stripped |
| `slug` | URL routing |
| `sku` | Unique identifier |
| `permalink` → `wpUrl` | 301 redirects + debug reference to live site |
| `_links.self.href` → `productUrl` | Single debug reference to WooCommerce API |
| `description` | Stripped to plain text |
| `descriptionImgs[]` | Extracted image URLs from description HTML |
| `short_description` → `summary[]` | Parsed from `<li>` tags, plain text fallback |
| `sale` | `on_sale == true AND sale_price < regular_price` |
| `regularPriceInCents` / `salePriceInCents` | String → integer |
| `averageRating` / `reviewCount` | Kept on products, dropped from variations |
| `inStock` | Binary — no low stock counts |
| `categories[]` | Objects with `{name, link}` — name uses slug value |
| `images[]` | `{src, name}` only — full res, no srcsets/thumbnails |
| `attributes[]` | Flat term slug array |
| `variations[]` | `{attribute: [...], variation: {...}}` |
| `weight`, `dimensions` | String → number, units hardcoded |

### Dropped
`id`, `parent`, `type`, `price_html`, `price_range`, `tags`, `brands`, `grouped_products`, `has_options`, `is_purchasable`, `is_on_backorder` (never true), `low_stock_remaining`, `stock_availability`, `sold_individually`, `is_password_protected`, `extensions`, `add_to_cart`, `_links`, `formatted_weight`, `formatted_dimensions`, all currency fields, image srcsets/thumbnails/alt

---

## Bucketing Logic

Products that pass one check go to that file — priority order matters:

1. `regular_price == 0` → `zero_price_products_shape.json` (8) — internal/service products
2. SKU is null or contains a space → `invalid_sku_products_shape.json` (22)
3. Any variation missing both stub value AND variation string → `missing_attributes_products_shape.json` (7)
4. No images → `no_image_products_shape.json` (3)
5. Duplicate cleaned name → `duplicate_name_products_shape.json` (6)
6. No weight → `no_weight_products_shape.json` (12)
7. Everything else → `clean_products_shape.json` (589)

Total: 647 — adds up exactly.

---

## Image Download

- **Folder structure:** `images/{shape}/products/{sku}/`, `variations/{sku}/`, `descriptions/{sku}/`
- **Keyed by SKU** — falls back to slug if SKU missing. The folder is the identifier, the filename inside doesn't matter
- **`url_map.json`** per shape folder — maps old bikershades.com URL → local path. Serves as both checkpoint (skip already-downloaded URLs) and translation layer for the upload step
- **`skipped.json`** at `images/` root — all 404'd URLs grouped by shape folder
- **Crash-safe** — saves after every individual download. Re-running skips cached files and retries failures
- **Description images are shared** — 40 unique URLs referenced 990 times. Upload step deduplicates by checking if a local path has already been uploaded before re-uploading

---

## Database Schema

Multi-brand Supabase/Postgres. Source of truth: `migration/db/001_initial_schema.sql`.

### Architecture
- 3 frontends (one per brand), 1 shared backend, 1 Supabase DB
- Frontend → Backend API → Supabase (frontend never hits Supabase directly)
- Backend uses `service_role` key — bypasses RLS for all operations
- RLS enabled on all tables as a safety net

### Tables
- `brands` — `id`, `name`
- `categories` — `id`, `brand_id`, `name`
- `products` — full product fields, `attributes text[]`, `summary text[]`
- `product_categories` — junction table (many-to-many)
- `variations` — full variation fields, `variation text[]` (attribute combo)
- `product_images` — `product_id`, `src`, `name`, `sort_order`
- `variation_images` — `variation_id`, `src`, `name`, `sort_order`
- `description_images` — nullable `product_id` OR `variation_id` (check constraint enforces exactly one), `src`, `name`, `sort_order`
- `admins` — `user_id` → `auth.users(id)`

### Auth
- Supabase Auth handles login and JWT
- No public signup — Supabase's `/auth/v1/signup` endpoint is publicly accessible so being in `auth.users` alone isn't enough. `admins` table is the real gate
- Admin flow: login → JWT → backend validates → checks `admins` table → `service_role` CRUD

---

## Up Next

- [ ] Upload images to storage (Supabase Storage / S3 / Cloudinary)
  - Skip images not in `url_map.json` (404'd — drop from DB entirely)
  - Dedup by local path before uploading (description images shared across products)
  - Write new `url_map.json` with hosted URLs swapped in
- [ ] Write import script — insert clean products into Supabase
- [ ] Manually review and fix other buckets (invalid SKU, no weight, duplicates, missing attributes)
- [ ] Build bikershades frontend (Next.js)
- [ ] Build shared backend (admin-api-mcp)
