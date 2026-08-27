# WooCommerce → Supabase Migration — Learnings (June 3, 2026)

_Built: May–June 2026_

---

## What I Built

A multi-brand WooCommerce → Supabase migration pipeline. Two brands run through it: **proSPORT Sunglasses** and **Sunglass Monster**. Each brand has its own WooCommerce store; the pipeline pulls product + variation data from the authenticated admin REST API and inserts it into a shared Supabase Postgres database with shared table structure but isolated by `brand_id`.

The pipeline is file-based and staged — each stage writes JSON to disk, and the next stage reads from the previous stage's output. This makes it easy to inspect, re-run from any point, and debug.

---

## Pipeline Architecture

### Stage order

```
fetch_products → slim_products → validate_products
fetch_variations → slim_variations → validate_variations
reshape_products → reshape_variations → create_items
{brand}_categorize_items → download_items → upload_items → import_items
```

### Why file-based and staged?

- Each stage is independently inspectable — you can `jq` any file at any point
- Re-running one stage doesn't require re-fetching from the API
- Flagged records are dropped at validate time and excluded from all downstream stages — no silent bad data flowing forward
- Crash-safe for slow operations: download and upload use a `*_map.json` that acts as a resume checkpoint

### Why a separate slim step?

Raw WooCommerce API responses are huge — hundreds of fields per product, most irrelevant. Slimming early makes every downstream script faster to read and debug, and it makes the shape of the data explicit.

### Why a separate reshape step?

Validate checks fields are present and valid. Reshape normalizes them to the format the DB expects: HTML stripped from descriptions, prices converted to cents, attributes name-cleaned, image objects built, stock nullified for variable products. Two separate concerns.

### Why brand-specific categorize script?

Category mapping is subjective and brand-dependent — the WooCommerce category names differ per brand and the taxonomy you want in the new DB is a design decision. Keeping it brand-specific means the logic is transparent and easily changed without touching shared code.

---

## API Learnings

### Admin REST API vs Store API

Used authenticated admin REST API (`/wp-json/wc/v3/`) with consumer key + secret (HTTP Basic Auth). The old BikerShades pipeline used the unauthenticated Store API — it was lossy.

| Field | Store API | Admin REST API |
|-------|-----------|----------------|
| Stock | `is_in_stock` bool | `stock_quantity` int + `stock_status` string |
| Prices | Inside `prices{}` object | Top-level strings (`regular_price`, `sale_price`) |
| Variation images | `images[]` array | `image` singular object |
| Data completeness | Partial | Full |

### Variation image shape

The admin API returns `image` (singular object `{id, src, name, alt}`) not `images[]`. Reshape normalizes this to `images[]` for consistency. At most one image per variation on these brands.

### WooCommerce slug validation

WooCommerce slugs diverge from `slugify(name)` — apostrophes get dropped, long names get truncated, WC adds `-2` suffixes for duplicates. The original pipeline compared `slugify(name) == slug` and flagged mismatches. Removed that check; just verify non-empty string.

### Attribute shape

Variation attributes come as `[{id, name, slug, option}]` where `option` is the term name (not slug). Product attributes come with `options[]` which can be empty — `create_items.py` fills them from variation data.

---

## Data Quality Learnings

### Variable product stock is misleading

WooCommerce aggregates `stock_quantity` at the product level for variable products, but it's not meaningful — the real stock is per-variation. Pipeline sets `stock=null` for all variable products. Same reasoning applies to `weight` and `dimensions`: if a product has variations with different weights or sizes, the product-level value is ambiguous or wrong.

### Partial dimensions

Some products have only some of length/width/height filled. All-or-nothing rule: if the set is partial, nullify all three and print an `[INFO]` line. Doesn't block the pipeline — surfaces data quality without hard-failing.

### Attribute name collisions

WooCommerce stores like to have both `Lens Type for Filter` and `Filter by Lens Type` on the same product. After cleaning (strip "for Filter", "Filter by ", "Choose ", etc.), both become `Lens Type` — a collision. Validate time catches this by simulating `clean_attr_name` and flagging products where two attribute names map to the same cleaned name.

The fix is `ATTR_OVERRIDES` — a brand-keyed dict in each reshape script that maps specific raw names to canonical names before the generic rules run.

### WooCommerce category data is messy

Categories come as flat `[{id, name, slug}]` arrays. They include junk like `Uncategorized` and `Big and Tall`. The categorize script:
1. HTML-decodes names (WC sometimes HTML-encodes `&` → `&amp;`)
2. Renames via `RENAME_MAP` (old WC name → new canonical name + slug, or `None` to drop)
3. Keeps only leaf slugs (WC sometimes includes both parent and child in the array)
4. Falls back to `HARDCODED` dict for products that had no usable WC category

---

## Schema Design Learnings

### Variable products: stock is nullable

`products.stock int` — nullable. Variable products get `null`. Simple products get a real integer.

### Sort order on images

Every image object carries a `sortOrder` field (1-indexed) set during reshape. The DB has `sort_order int` on `product_images`, `variation_images`, `description_images`. This means the display order is stable across re-imports and the frontend can `ORDER BY sort_order` without guessing.

### Sort order on categories

Every node in every category path gets a `sortOrder` (1-indexed, resets per parent). Keyed by slug in a flat `SORT_ORDER` dict in the categorize script — same slug always gets the same sort order, no matter which product references it. DB has `sort_order int not null default 1` on `categories`.

### Import as pure insertion

`import_items.py` does no field generation. Every value it inserts comes directly from the categorize-items JSON. No `enumerate` to generate `sort_order`, no inline `None` for units, no `.get("stock", 1)` fallbacks. If the data is wrong, fix it upstream. This makes import a thin translation layer.

### `weightUnit` / `dimensionUnit`

Currently `null` for both brands — neither populates these in WooCommerce. They live in the reshape output as null fields so import can read them without special-casing. When a brand eventually populates them, only reshape changes.

---

## QA Pattern

### Audit vs Verify

- **Audit** — describes the data: field coverage, distribution, counts, edge cases. Tells you what you have. No pass/fail.
- **Verify** — asserts correctness: cross-checks output against source, confirms no records dropped, checks field invariants. Has explicit PASSED / FAILED output.

### What verify checks

- Every source record present in output (and no extras)
- Fields not modified that shouldn't be
- Invariants per field type (e.g., `sortOrder` sequential from 1, dims all-or-nothing, variable weight is null)
- Cross-stage correctness (e.g., verify_create_items checks variation data matches reshape-variations source)

---

## Crash Safety Pattern

Download and upload are the two stages that talk to external services and take a long time. Both scripts use a `*_map.json` as a resume checkpoint:

- **download**: `download_map.json` maps `original_src → local_path`. On restart, skip any src already in the map.
- **upload**: `upload_map.json` maps `local_path → supabase_url`. On restart, skip any local_path already in the map.

Both commit entries to the map immediately after each successful operation, so a crash mid-run resumes from the last successful item.

---

## Category Tree Design

### Structure

Every category path is an array of node objects:
```json
[
  {"level": 1, "name": "Lens Color", "slug": "lens-color", "sortOrder": 2},
  {"level": 2, "name": "Polarized",  "slug": "polarized",  "sortOrder": 1}
]
```

A product can have multiple paths (multi-leaf). `sortOrder` on root nodes determines navbar button order; `sortOrder` on leaf nodes determines order within a dropdown.

### `ensure_category` in import

Builds the category tree lazily using a `(parent_id, slug)` key. First time a path is seen, each node is inserted; subsequent products sharing a node reuse the existing ID. This means categories are created exactly once regardless of how many products reference them.

---

## Multi-Brand Pattern

### `BRAND` env var

`.env` has `BRAND=prosport` or `BRAND=monster`. All scripts call `load_brand()` which reads the env var, looks it up in `BRAND_CONFIG`, and returns the slug + credentials. Switching brands is one line change.

### Data isolation

All output scoped to `data/{slug}/` — multiple brands can coexist on disk without collision.

### Brand-specific scripts

Only `{brand}_categorize_items.py` is brand-specific. Everything else is shared. The categorize script is brand-specific because: category rename maps are brand-specific, the category tree structure is brand-specific, and hardcoded fallback assignments are brand-specific.

---

## Counts

### proSPORT (run-2, in progress)

| Stage | Count |
|-------|-------|
| Fetched products | 171 |
| Validated products | 169 (2 flagged) |
| Validated variations | 1,740 (18 flagged: 17 SKU spaces, 1 no price) |
| create-items | 166 (3 flagged) |
| categorize-items | 166, 197 leaf assignments |
| Categories | 12 (4 roots, 8 leaves) |
| Download / upload / import | Pending |

### Sunglass Monster (in progress)

| Stage | Count |
|-------|-------|
| Fetched products | 277 |
| Validated products | 267 (10 flagged) |
| Validated variations | 2,673 (2 flagged) |
| create-items | 265 (2 flagged), 2,646 variations embedded |
| categorize-items | Pending |

---

## Things I'd Do Differently

- **Start with the admin API** — the Store API (unauthenticated) is a dead end for real migrations. Always use the admin API from day one.
- **All-or-nothing dimensions from the start** — partial dims silently passed through the old pipeline and became nulls at import without any signal. The INFO print + upstream nullification is cleaner.
- **Sort order belongs in the data, not the import script** — generating sort order via `enumerate` in import is fragile. Setting it in reshape means every QA stage can verify it, and import is a dumb reader.
- **Validate at every stage boundary** — the verify scripts are the best debugging tool. Running them immediately after every stage would have caught issues earlier in the old pipeline.
