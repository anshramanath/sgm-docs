# Bikershades Migration Progress (May 29, 2026)

## Overview

Migrating product data from bikershades.com (WordPress/WooCommerce) into a new app/database.

**Source:** `https://bikershades.com/wp-json/wc/store/products`
**Working directory:** `/Users/anshramanath/Desktop/bikershades-migration/migration/`

---

## Current File Structure

```
bikershades-migration/
  bikershades_migration_guide.md   ← original migration guide
  PROGRESS.md                      ← this file
  migration/
    fetch_products.py              ← step 1: fetch all products
    fetch_variations.py            ← step 2: fetch all variations
    reshape_products.py            ← step 3: reshape into clean schema
    upload_images.py               ← step 5: upload local images to Supabase Storage
    import_products.py             ← step 6: insert clean products into Supabase DB
    products/
      products.json                ← raw WooCommerce backup (647 products)
      variations/
        variations.json            ← per-variation data (6,064 variations)
    reshaped/
      clean_products_shape.json         ← 589 clean products
      zero_price_products_shape.json    ← 8 internal/service products
      invalid_sku_products_shape.json   ← 22 products with invalid SKUs
      missing_attributes_shape.json     ← 7 products with unrecoverable variation attributes
      no_image_products_shape.json      ← 3 products with no images
      duplicate_name_products_shape.json← 6 products with duplicate names
      no_weight_products_shape.json     ← 12 products with no weight
    images/
      clean-products/
        url_map.json               ← old URL → local path mapping + checkpoint
        upload_map.json            ← local path → Supabase Storage URL mapping + checkpoint
        products/{sku}/            ← product images per SKU
        variations/{sku}/          ← variation images per variation SKU
        descriptions/{sku}/        ← description-embedded images per product SKU
      zero-price/
      invalid-sku/
      missing-attributes/
      no-image/
      duplicate-name/
      no-weight/
```

---

## Steps Completed

### Step 1 — Fetch all products
**Script:** `fetch_products.py`
**Result:** `products.json` — 647 products across 7 pages (per_page=100)

### Step 2 — Fetch all variations
**Script:** `fetch_variations.py`
**Result:** `variations.json` — complete (6,064/6,064 variations, 486/486 parents)
**Total variations:** 6,064 across 486 variable products

### Step 4 — Download images
**Script:** `download_images.py`
**Result:** 2,785 images downloaded across all 7 shape folders

- Folder structure: `images/{shape}/products/{sku}/`, `variations/{sku}/`, `descriptions/{sku}/`
- Folders keyed by SKU (falls back to slug if SKU missing)
- `url_map.json` per shape folder: maps old bikershades.com URL → local file path. Acts as both checkpoint and the translation layer for future upload step
- Some 404s logged and skipped — 86 variation images + 10 description images from 2018 that no longer exist on the server. Zero product-level images lost.
- All failed URLs written to `images/skipped.json` keyed by shape folder
- Crash-safe: checkpoints after every individual download via `url_map.json`

### Step 3 — Reshape products
**Script:** `reshape_products.py`
**Result:** 7 output files in `reshaped/` — 589 clean, 647 total split by issue type

Key reshape decisions applied:
- Descriptions stripped to plain text, embedded image URLs extracted into `descriptionImgs[]`
- `summary` parsed from `<li>` tags into string array, with plain text fallback
- `sale` computed from two checks: `on_sale == true AND sale_price < regular_price`
- `averageRating` normalized to float
- `categories` reshaped as `[{name, link}]` objects — name uses slug value, link preserves old WP category URL
- `attributes` reshaped as `[{name, terms[]}]` grouped by attribute type — e.g. `{name: "color", terms: ["black-clear", "blue-clear"]}`. 17 raw WooCommerce attribute names mapped to clean short keys. Stored as `jsonb` in DB for structured querying on category/listing pages without joining variations
- `variations` array: each entry `{ attribute: [...raw stub values], variation: {...full variation object} }`
- Variation `summary`, `averageRating`, `reviewCount` dropped — always empty/zero at variation level
- Names decoded from HTML entities and tags stripped
- Products split into separate files by issue type: zero price, invalid SKU, missing attributes, no images, duplicate names, no weight

---

## Data Analysis Findings

### Product types
| Type | Count |
|---|---|
| variable | 489 |
| simple | 157 |
| prescription | 1 |

### Key observations
- **75% of products are variable** — they have color/option attributes and variation IDs
- **Prices are strings in cents** — `"1999"` = $19.99
- **524/647 products are on sale** (~81%)
- **12 products have zero images**
- **3 products have no categories**
- **`stock_status` is null on every product** — actual stock is in `is_in_stock` (bool) and `stock_availability.text`

### Image distribution
Products range from 0 to 18 images each. Most have 2-7.

### Variations
The main products endpoint only returns variation IDs + attribute slugs — not per-variation images, price, or stock. A second pass is required to get full variation data.

Each variation has:
- `parent` — ID of the parent product
- `sku` — unique per color/option
- `images` — specific to this variation
- `prices` — can differ per variation
- `is_in_stock` / `stock_availability`
- `variation` — human-readable label e.g. `"Choose Color (REQUIRED): Blue Green Smoke"`

### Attributes / color swatches
Variable products have `attributes[].terms[]` — each term has a `name` and `slug`. These drive the color swatches shown on the live site.

---

## Scripts

### `fetch_products.py`
Paginates through the WooCommerce Store API and saves all products to `products.json`. Stops when an empty page is returned.

### `fetch_variations.py`
Fetches full variation data for every variation ID found in `products.json`. Saves results to `variations.json` keyed by parent product ID.

**Resume logic:** checkpoints after every individual variation. On restart, checks which variation IDs are already saved per parent and only fetches the missing ones — safe to crash and resume at any point.

---

## Decisions Made

- **Fetch full variation data** (rather than skip) — needed for per-color images, stock, and SKUs
- **0.3s sleep between requests** — polite to the live server, avoids rate limiting
- **Checkpoint per variation** — crash-safe, resumes exactly where it left off
- **Descriptions stripped to plain text** — both product and variation descriptions stripped of HTML. Embedded image URLs extracted separately into `descriptionImgs[]`
- **`descriptionImgs[]`** — array of image URLs extracted from description HTML. 587 products have them (990 total URLs), 4 variations have them. Must be downloaded and rehosted
- **`on_sale` computed from two checks** — `on_sale == true AND sale_price < regular_price`. Owner intent (`on_sale`) + price validation both must be true. `sale_price` can exist without the product being actively on sale (owner may have it staged for future use)
- **Sale page is manually curated** — the site's sale page is not an automatic filter of all `on_sale` products. For the new app, use the `on_sale` flag for badges/pricing; build the sale page view with whatever logic fits
- **`permalink` kept as `wpUrl`** — useful for 301 redirects from old Google-indexed URLs and for debugging by comparing against the live site
- **Drop entire currency block** — all 647 products are USD only. `currency_symbol`, `currency_code`, `currency_minor_unit`, separators, prefix/suffix are all redundant. Frontend hardcodes `$` and cents division
- **Drop `price_html`** — pre-rendered WooCommerce HTML for price display, redundant when you have raw price integers
- **Drop `tags` and `brands`** — 4/647 products have tags, 0 have brands. Effectively unused
- **Drop `srcset`, `sizes`, `thumbnail_srcset`, `thumbnail_sizes` from images** — keep only `src` (full res original) and `name`. Thumbnails/srcsets generated by CDN/storage. `alt` dropped (5% populated)
- **`attributes` as flat slug array** — all attribute term slugs kept regardless of `has_variations`. Frontend ignores what it doesn't need
- **Variations excluded from pagination by design** — WooCommerce only returns parent products via pagination to keep the catalog clean. Variations fetched separately via `/products/{id}`
- **Drop `_links` block entirely** — API metadata pointing back to old WordPress endpoints, useless post-migration
- **Keep one `productUrl`** — store `https://bikershades.com/wp-json/wc/store/products/{id}` per product as a single debug reference. Keeps DB clean while giving a direct way to inspect the original data if anything looks wrong

---

## Reshape Decisions

### Product fields
| WooCommerce field | Renamed to | Notes |
|---|---|---|
| `name` | `name` | |
| `slug` | `slug` | for URL routing, already unique |
| `sku` | `sku` | |
| `permalink` | `wpUrl` | for 301 redirects + debugging |
| `_links.self.href` | `productUrl` | single debug reference back to WooCommerce API |
| `description` | `description` | stripped to plain text |
| `description` (img tags) | `descriptionImgs[]` | extracted image URLs for download/rehost |
| `short_description` | `summary` | parse `<li>` tags into string array, plain text fallback |
| `on_sale` | `sale` | only true if `on_sale == true AND sale_price < regular_price` |
| `prices.regular_price` | `regularPriceInCents` | string → integer |
| `prices.sale_price` | `salePriceInCents` | string → integer |
| `average_rating` | `averageRating` | normalized to float, seed data for new review system |
| `review_count` | `reviewCount` | seed data for new review system |
| `is_in_stock` | `inStock` | |
| `categories[].slug` | `categories[]` | flat array of slugs only — for filtering |
| `images[].src` | `images[].src` | full res original only |
| `images[].name` | `images[].name` | |
| `attributes[].terms[].slug` | `attributes[]` | flat array of term slugs, all attributes kept regardless of `has_variations` |
| `variations` (merged with variations.json) | `variations[]` | each entry: `{ attribute: ["black-clear", "1-00"], variation: {...full variation object} }` — `attribute` uses raw stub values as-is |
| `weight` | `weight` | string → number |
| `dimensions` | `dimensions` | all strings → numbers |
| — | `weightUnit` | hardcoded `"oz"` |
| — | `dimensionUnit` | hardcoded `"in"` |

### Variation fields
| WooCommerce field | Renamed to | Notes |
|---|---|---|
| `name` | `name` | |
| `slug` | `slug` | |
| `sku` | `sku` | |
| `variation` | `variation` | raw stub attribute values array e.g. `["black-clear", "1-00"]` |
| `permalink` | `wpUrl` | |
| `_links.self.href` | `productUrl` | |
| `description` | `description` | stripped to plain text |
| `description` (img tags) | `descriptionImgs[]` | extracted image URLs (only 4 variations have these) |
| `on_sale` | `sale` | same two-check logic as product |
| `prices.regular_price` | `regularPriceInCents` | string → integer |
| `prices.sale_price` | `salePriceInCents` | string → integer |
| `is_in_stock` | `inStock` | |
| `images[].src` | `images[].src` | |
| `images[].name` | `images[].name` | |
| `weight` | `weight` | string → number |
| `dimensions` | `dimensions` | all strings → numbers |
| — | `weightUnit` | hardcoded `"oz"` |
| — | `dimensionUnit` | hardcoded `"in"` |

**Dropped from variations only:** `summary` (always empty), `averageRating` (always 0), `reviewCount` (always 0), `categories`, `attributes`

### Fields to drop
`id` (new DB generates its own), `parent` (redundant — already in `productUrl`), `type` (inferable from whether `variations` is non-empty), `variation`, `price_html`, `price_range`, `tags`, `brands`, `grouped_products`, `has_options`, `is_purchasable`, `is_on_backorder` (never true across all products and variations), `low_stock_remaining` (keep stock binary), `stock_availability` (both `text` and `class`), `sold_individually`, `is_password_protected`, `extensions`, `add_to_cart`, `_links`, `formatted_weight`, `formatted_dimensions`, all currency fields, `srcset`/`sizes`/`thumbnail`/`thumbnail_srcset`/`thumbnail_sizes` from images, image `alt` (only 5% populated)

### Variations array (merged in)
Each variation stub in `products.json` looks like:
```json
{ "id": 65388, "attributes": [{ "name": "Choose Color", "value": "black-clear" }, { "name": "Choose Power", "value": "1-00" }] }
```

In the reshape, replace `id` with the full variation object from `variations.json`, and simplify `attributes` to just values (drop `name` — order matches parent attribute keys, and the variation string on the full object has names if ever needed):
```json
{
  "attribute": ["black-clear", "1-00"],
  "variation": { ...full variation object, `variation` field is raw stub values array... }
}
```

Whether a product is variable is determined solely by whether `variations` is non-empty. `type` and `has_variations` are not used as source of truth.

**38 empty `variation` strings** — resolved by using `attributes` array from the variation stub in `products.json` as the source of truth for attribute values. No fallback parsing needed.

**9 simple products with `has_variations: true` attributes** — WooCommerce misconfiguration. Resolve naturally since they have no variations. No special handling needed.

---

## Known Issues (Priority Order)

1. ~~**Filter out zero-price products**~~ — resolved. 8 products filtered in reshape.

2. ~~**`short_description` parsing fallback**~~ — resolved. Plain text fallback implemented in `reshape_products.py`.

3. ~~**990 description image URLs across 587 products + 4 variations**~~ — downloaded. Some 404 (old deleted files). `url_map.json` per shape tracks what was successfully saved.

4. **86 variation images + 10 description images 404** — old files deleted from WordPress server (~2018 era). Not recoverable. Upload step will drop image objects whose `src` wasn't downloaded rather than inserting broken URLs. Frontend falls back to parent product images for variations with no images.

5. **56 variations with no images** — low priority. Frontend falls back to parent product images naturally.

---

## Reshaped Problems Review

Detailed manual review notes now live in `reshaped-problems/`.

### Review files

- `reshaped-problems/diagnosis.md` — high-level summary of all reviewed buckets
- `reshaped-problems/zero_price_review.md`
- `reshaped-problems/no_weight_review.md`
- `reshaped-problems/no_image_review.md`
- `reshaped-problems/invalid_sku_review.md`
- `reshaped-problems/missing_attributes_review.md`
- `reshaped-problems/duplicate_name_review.md`

### Current bucket status

| Bucket | Count | Status | Direction |
|---|---:|---|---|
| zero_price | 8 products | reviewed | mostly owner review; mix of excludes, form/workflow items, unavailable products, and at least one real product that needs price |
| no_weight | 12 products | reviewed | mostly owner review; some real products just need confirmed weight |
| no_image | 3 products | reviewed | ask owner for real images; only use placeholders with explicit approval |
| invalid_sku | 22 products | reviewed | mostly owner review; many are real products that need canonical SKU cleanup |
| missing_attributes | 7 products | built / owner escalation | take entire bucket to owner; variation option metadata is not safe to reconstruct blindly |
| duplicate_name | 6 products | built / owner escalation | take entire bucket to owner; distinct products collapsed to duplicate reshaped names |

### High-level findings

- **Owner review is now the main blocker** for `duplicate_name`, `missing_attributes`, and most of `invalid_sku`
- **`missing_attributes` means both sources are bad** for the affected variations: the reshaped variation `attribute` field is empty and the nested Woo variation `variation` field is also empty
- **38 variations are affected in `missing_attributes`** across 7 products:
  - `Magnum – Clear` → 2 / 36
  - `Locs Impala 91025` → 4 / 4
  - `Seaway` → 2 / 4
  - `Aruba Polarized Sunglasses` → 1 / 10
  - `Taxi Polarized Bifocal [Fits MEDIUM -XLARGE Heads]` → 1 / 22
  - `Starlet Over Sized Fashion Bifocal` → 16 / 16
  - `Finish Line Bifocal [Fits MEDIUM – XLARGE Heads]` → 12 / 12
- **`duplicate_name` has 6 products** but only 2 duplicated reshaped names:
  - `BikerArmour Angel Chrome with Rhinestones [MED – LG]`
  - `BikerArmour Barricade [MED – XLG]`
- **`no_image` is fully triaged** and mostly comes down to recovering missing media from the owner
- **`zero_price` and `no_weight` contain the clearest “fix and migrate” candidates** once the owner answers a short list of questions

### Practical next step

After the owner review, the likely execution order is:

1. Resolve `zero_price` keep/exclude/price questions
2. Fill missing weights for products we intend to keep
3. Clean canonical SKUs for valid products in `invalid_sku`
4. Recover missing images where available
5. Decide how to handle duplicate names and unrecoverable variation metadata

---

## Database Schema

Multi-brand Supabase/Postgres schema. Auth handled by Supabase Auth (`auth.users`), admin access gated by `admins` table.

### Tables

| Table | Key fields |
|---|---|
| `brands` | `id`, `name` |
| `categories` | `id`, `brand_id`, `name` |
| `products` | `id`, `brand_id`, `name`, `slug`, `sku`, `wp_url`, `product_url`, `description`, `summary[]`, `attributes[]`, `sale`, `regular_price_cents`, `sale_price_cents`, `average_rating`, `review_count`, `in_stock`, `weight`, `weight_unit`, `length`, `width`, `height`, `dimension_unit` |
| `product_categories` | `product_id`, `category_id` (junction) |
| `variations` | `id`, `product_id`, `name`, `slug`, `sku`, `variation[]`, `wp_url`, `product_url`, `description`, `sale`, `regular_price_cents`, `sale_price_cents`, `in_stock`, `weight`, `weight_unit`, `length`, `width`, `height`, `dimension_unit` |
| `product_images` | `id`, `product_id`, `src`, `name`, `sort_order` |
| `variation_images` | `id`, `variation_id`, `src`, `name`, `sort_order` |
| `description_images` | `id`, `product_id`, `src`, `sort_order` |
| `admins` | `user_id` → `auth.users(id)` |

### Key decisions
- `brands` is just `id` + `name` — slug derivable in code
- `categories` is just `id` + `brand_id` + `name` — 63 unique category pages exist on the site, but brand/lens-type/head-size filtering comes from `attributes[]` not categories. Categories used only for top-level nav grouping
- `attributes` is `jsonb` — stores `[{name, terms[]}]` grouped by attribute type for efficient listing-page queries without joining variations
- `variation[]` on variations table is `text[]` — flat slug array of the specific combo e.g. `["black-clear", "1-00"]`
- `summary[]` is `text[]` — parsed from `<li>` tags
- Dimensions stored as flat columns (`length`, `width`, `height`) not jsonb
- Separate image tables for product, variation, and description images — all have `src`, `name`, `sort_order`
- `description_images` has nullable `product_id` and `variation_id` with a check constraint enforcing exactly one is set — a description image belongs to either a product or a variation, not both
- `description_images.name` derived from filename in the import script (last path segment of `src`)
- No signup — `admins` table gates admin access since Supabase Auth signup endpoint is publicly accessible
- RLS enabled on all tables, frontend never hits Supabase directly — goes through backend API with `service_role` key
- Source of truth for schema: `migration/db/001_initial_schema.sql`

---

## Up Next

- [x] Wait for `variations.json` to complete
- [x] Write `reshape_products.py` — normalize WooCommerce schema into clean app schema
- [x] Write `download_images.py` — download all product/variation images locally (2,785 images)
- [x] Decide database schema — finalized (see below)
- [x] Upload images to Supabase Storage — 2,559 unique images uploaded to `product-images` bucket (public). 1 timed out on first run, retried successfully. `upload_map.json` written: `local_path → storage_url`.
- [x] Write import script — `import_products.py`. Wipes DB (TRUNCATE brands CASCADE), inserts in FK order: brands → categories → products → product_categories → variations → product_images → variation_images → description_images. Resolves image srcs via `url_map + upload_map` chain; drops any image not in both maps.
- [x] Run import — 589 products, 63 categories, 1 brand inserted successfully. Exit code 0.
- [ ] Manually review and fix flagged buckets (invalid SKU, no weight, duplicates, missing attributes, zero price, no image)
