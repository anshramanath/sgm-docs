# Bikershades Migration — Build Notes (May 31, 2026)
_Last updated: 2026-05-31_

Everything learned and built across both pipeline runs. Decisions, tradeoffs, bugs fixed, and patterns that emerged.

---

## What We Built

A 7-step Python pipeline that migrates 647 WooCommerce products into a normalized Supabase/Postgres schema. Two full runs completed. The pipeline is crash-safe, resumable, and fully verified at every step via paired audit/verify QA scripts.

---

## The Pipeline

### Step 1 — Fetch Products (`fetch_products.py`)
Paginates the WooCommerce Store API (`/wp-json/wc/store/products`) at 100/page, stops on empty page. Saves to `items/products.json`.

**Bug fixed:** Script was creating a `products/` directory instead of `items/`. Fixed `mkdir` path.

### Step 2 — Fetch Variations (`fetch_variations.py`)
The main products endpoint only returns variation stubs (ID + attribute slugs). Full variation data (price, stock, images, SKU) requires a second pass to `/products/{id}`.

Checkpoint pattern: after each variation, write to `items/variations.json`. On resume, check which IDs are already present and skip them. Safe to kill at any point.

**Bug fixed:** Output was saved to `items/variations/variations.json`. Fixed to `items/variations.json` (flat, matches what reshape expects).

### Step 3 — Reshape (`reshape_items.py`)
The heaviest script. Normalizes raw WooCommerce data into the app schema and buckets problems by type.

Key decisions:
- **Price model:** Variable products derive `min_price_cents`/`max_price_cents` from variation `regular_price_cents` (not product-level price — that's stale). Simple: min = max = product regular price. All price logic here, not in import.
- **Sale logic:** Variable = any variation has `sale = true`. Simple = `sale_price > 0 AND sale_price < min_price`. Ignores WooCommerce `on_sale` flag entirely — price comparison is source of truth.
- **`sale_price_cents` nullable on products:** Null for variable products (use variation prices). Non-null for simple.
- **Stock:** `is_in_stock: true → 1`, `false → 0`. Int so owner can update to real quantity later; frontend treats `> 0` as in stock.
- **Ratings dropped:** `average_rating` and `review_count` removed. App builds its own review system.
- **Descriptions:** HTML stripped to plain text. Image URLs extracted into `descriptionImgs[]`.
- **Attributes:** Grouped by type into `{name, terms[{name, slug}]}`. Power term slugs normalized (`1-00` → `1.00` WooCommerce inconsistency).
- **38 empty variation strings:** Use `attributes` array from product stub as source of truth; don't try to parse the variation label string.

Buckets (problem items separated, not dropped):
| Bucket | Count | Reason |
|--------|------:|--------|
| `shaped_items.json` | 586 | ✅ Clean |
| `invalid_sku_items.json` | 22 | SKU null or contains space |
| `no_weight_items.json` | 14 | Null or zero weight |
| `zero_price_items.json` | 8 | Price is zero |
| `missing_attributes_items.json` | 7 | Variation option data missing from API |
| `duplicate_name_items.json` | 6 | Same name as another item |
| `no_image_items.json` | 3 | No images at all |
| `multiple_variation_images_items.json` | 1 | WooCommerce misconfiguration |

### Step 4 — Recategorize (`recategorize_items.py`)
WooCommerce category links are messy — products can have parent + child category links, duplicate paths, and may be sole occupants of a leaf. This script:
1. Applies a drop list (remove specific slugs) and merge map (redirect slug → new slug)
2. Iteratively removes single-item leaves → `single_leaf_items.json`
3. Strips non-leaf paths (if a product has both a parent and child category, keep only the child)
4. Anything with no remaining categories → `no_category_items.json`

Result: 8 trees, max depth 3, products only at leaf nodes.

Category shape: `[[{level, name, slug}]]` — outer array = multiple paths, inner array = root-to-leaf path.

**Key insight:** Storing products only at leaves means nav never has to disambiguate between "this node is a category page" vs "this node is a product page."

### Step 5 — Download Images (`download_images.py`)
Collects all image URLs across `images[].src`, `descriptionImgs[]`, and variation `images[].src` for the 581 categorized items. Deduplicates by URL (same image referenced across products downloaded once).

Image naming: path after `/uploads/` or `/gallery/` with `/` → `_` (e.g. `2024_05_image.jpg`). Globally unique by design — no collisions.

Checkpoint: after each download, write URL → local path to `url_map.json`. On resume, skip URLs already in map.

Failed downloads tracked in `failed.json` as `{failed_to_download: [{sku, src}], collisions: []}`.

**30 permanent 404s:** All are 2018-era variation/description images deleted from the server. No product loses all its images.

**Earlier collision problem:** First naming scheme used only the filename end (e.g. `image.jpg`). Gallery URLs with `?i=` params caused collisions. Fixed by using the full path after `/uploads/` or `/gallery/` as the filename — these paths are date-prefixed and globally unique.

### Step 6 — Upload Images (`upload_images.py`)
Uploads every local file to the `bikershades` Supabase Storage bucket (public, flat structure). Checkpoint: after each upload, write local path → Supabase URL to `upload_map.json`.

**Crash recovery patch:** If killed mid-upload, the file lands in Supabase but the map write may not happen. On retry, Supabase returns 409 Duplicate. Original code counted this as a failure, leaving the file permanently missing from the map. Fixed: catch `"Duplicate"` in error string, call `get_public_url()` directly, write to map. Recovered 1 file in run 2.

### Step 7 — Import (`import_items.py`)
Reads `recategorized/categorized_items.json` + both image maps. Inserts in FK order:

1. Brand (BikerShades)
2. Category tree — built lazily with `ensure_category(path)` using `(parent_id, slug)` dedup key. Recursively walks each root-to-leaf path.
3. Products — `min_price_cents`, `max_price_cents`, `sale_price_cents` (nullable), `stock`, no ratings.
4. `product_categories` junction — ON CONFLICT DO NOTHING.
5. Variations — `regular_price_cents`, `sale_price_cents` (not null), `stock`, `image_src`/`image_name` inline.
6. Product images — resolved via `url_map → upload_map` chain. Dropped if either lookup fails (no broken URLs in DB).
7. Description images — same resolution chain.

---

## QA System

Every pipeline step has two paired scripts:

| Type | Purpose |
|------|---------|
| `audit_*` | Internal validity — checks the output file/DB for correctness, completeness, and data quality |
| `verify_*` | Cross-reference — compares output against source to confirm nothing was lost or mutated |

Scripts print a reminder to update the corresponding doc in `docs/`. Results are written as markdown summaries, not raw output.

### Scripts
| Script | Checks |
|--------|--------|
| `audit_shaped_items.py` | Required fields present, price/stock/sale invariants, 0 or 1 stock values |
| `verify_shaped_items.py` | All 647 raw products accounted for, no duplicates, no data loss |
| `audit_categorized_items.py` | Leaf-only assignments, valid slugs, path integrity, tree stats |
| `verify_categorized_items.py` | 581+2+3=586, no dropped/merged slugs survived, fields unchanged |
| `audit_downloaded_images.py` | url_map coverage, files on disk, failed count, coverage by type |
| `verify_downloaded_images.py` | All URLs in map or failed, no URL unaccounted for, no product imageless |
| `audit_uploaded_images.py` | upload_map coverage, URL validity, no missing files |
| `verify_uploaded_images.py` | Full chain resolves for every downloaded file |
| `audit_import.py` | Row counts, brands, category depth, duplicate SKUs, products without images |
| `verify_import.py` | SKUs, field values, variation counts, category leaf slugs all match source |

---

## Schema Decisions

### Products table
- `min_price_cents` + `max_price_cents` — derived from variation prices for variable products. Denormalized for fast listing queries without joining variations.
- `sale_price_cents` nullable — null for variable products. Simple products store the WooCommerce sale price here.
- `stock int` — not boolean. Owner sets real quantities later; frontend gates on `> 0`.
- No `average_rating` / `review_count` — app builds its own review system.

### Variations table
- `regular_price_cents` + `sale_price_cents` both not null — source of truth for per-variation pricing.
- `image_src` + `image_name` inline — variations have at most 1 image. No separate table.
- `variation text[]` — flat slug array of the attribute combo (e.g. `["black-clear", "1.00"]`).

### Categories table
- Self-referential `parent_id` tree. Owner can create, rename, move categories through admin.
- `slug` for URL routing. WooCommerce link URLs not stored — hierarchy reconstructed at import from source paths.

### Image tables
- `product_images` + `description_images` — separate tables, both have `src`, `name`, `sort_order`.
- `description_images` has `product_id` and `variation_id` with a check constraint: exactly one must be set.

### Multi-brand
- `brands` table with `slug` matching the Supabase Storage bucket name.
- `categories` and `products` are brand-scoped.
- New brands get their own storage bucket.

### Auth / admin
- No public signup. `admins` table gates admin access via Supabase Auth.
- Frontend never hits Supabase directly — goes through Next.js backend with `service_role` key.

---

## Image Pipeline

```
bikershades URL
    ↓ url_map.json (URL → local path)
local file
    ↓ upload_map.json (local path → Supabase URL)
Supabase Storage URL
```

Both maps are checkpoints and the translation layer. If either lookup fails, the image is dropped at import — no broken URLs in DB.

Deduplication is free: `url_map.json` keys by full URL, so the same image referenced across 10 products is downloaded and uploaded once.

---

## Bugs Fixed

| Bug | Fix |
|-----|-----|
| `fetch_products.py` creating `products/` instead of `items/` | Fixed `mkdir` path |
| `fetch_variations.py` saving to `items/variations/variations.json` | Fixed output path to `items/variations.json` |
| Gallery URL collisions (`?i=` param on same image) | Use full path after `/uploads/` or `/gallery/` as filename |
| Duplicate collisions on retry | Deduplicate collision list on `(existing.src, tried.src)` before saving |
| Upload script leaving file permanently missing after kill | Catch 409 Duplicate, fetch public URL directly, write to map |
| `recategorized/` deleted on git pull | Was tracked in git; re-run `recategorize_items.py` |

---

## Catalog Numbers (Both Runs Match)

| Metric | Count |
|--------|------:|
| Raw products | 647 |
| Variable products | 486 |
| Simple products | 161 |
| Total variations | 6,064 |
| Clean shaped | 586 |
| Categorized (ready for import) | 581 |
| Flagged for review | 66 |
| Unique image URLs | 2,561 |
| Downloaded | 2,531 |
| Permanent 404s | 30 |
| Uploaded | 2,531 |
| Products in DB | 581 |
| Variations in DB | 5,649 |
| Categories in DB | 54 |
| Product images in DB | 2,618 |
| Description images in DB | 935 |

---

## Outstanding — Owner Review Needed

| Item | Count | Blocker |
|------|------:|---------|
| `invalid_sku_items.json` | 22 | Need canonical SKUs — mix of service items and real products |
| `no_weight_items.json` | 14 | Need weight values for real products |
| `zero_price_items.json` | 8 | Owner decides keep/price/drop — mix of Rx flows and real products |
| `missing_attributes_items.json` | 7 | Variation option data missing from API; owner checks live site |
| `duplicate_name_items.json` | 6 | Angel Chrome (×4) and Barricade (×2) — owner decides merge/rename/split |
| `no_image_items.json` | 3 | Owner provides or confirms missing assets |
| `no_category_items.json` | 2 | All category links dropped — owner recategorizes |
| `single_leaf_items.json` | 3 | Sole occupant of leaf — owner merges into parent or adds more products |
| **Total** | **66** | Share `reshaped-problems/treatment.md` |
