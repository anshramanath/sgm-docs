# Migration Learnings (June 3, 2026)
_Everything learned and built during the WooCommerce → Supabase migration_
_Last updated: 2026-06-03_

---

## What was built

A multi-brand WooCommerce → Supabase product migration pipeline. It fetches raw product and variation data from the WooCommerce admin REST API, cleans and normalizes it through a series of staged scripts, and imports clean structured data into a Supabase Postgres database with images in Supabase Storage.

Two brands migrated so far: **proSPORT Sunglasses** (complete) and **Sunglass Monster** (in progress).

---

## Pipeline architecture

### Staged, file-based pipeline
Each stage reads from the previous stage's output file and writes to its own output folder. No stage mutates upstream data. This means:
- You can re-run any single stage without touching others
- You can inspect the exact state of data at any point
- Failures are isolated — a crash in download doesn't corrupt reshape output

### Flagged vs passed split
Every validate and create stage produces two files: `products.json` (or `items.json`) for records that passed all checks, and `flagged.json` for records that failed. Flagged records are **excluded from all downstream stages** — they don't move forward. The flagged file is just a paper trail.

This means the pool of clean records shrinks at each stage. By the time you reach import, everything in the file has been validated end to end.

### Crash-safe download and upload
`download_items.py` and `upload_items.py` maintain a `download_map.json` / `upload_map.json` as a checkpoint file. On each image processed, it writes the result immediately. If the script crashes, re-running it skips already-completed images. Essential for large catalogs.

### QA pattern: audit + verify
Every stage has two QA scripts:
- **audit** — reads the output file and prints field coverage stats, distributions, and warnings. Lets you quickly see what the data looks like.
- **verify** — reads both the source and the output and cross-checks that every transformation happened correctly. Deterministic pass/fail.

Running both after each stage catches both data quality issues (audit) and transformation bugs (verify).

---

## WooCommerce API

### Admin REST API vs Store API
The old BikerShades pipeline used the Store API (unauthenticated). For this pipeline we switched to the **admin REST API** (`/wp-json/wc/v3/`) with HTTP Basic Auth (consumer key + consumer secret).

| | Store API | Admin REST API |
|--|-----------|----------------|
| Auth | None | HTTP Basic Auth |
| Stock | Binary `is_in_stock` bool | Real `stock_quantity` integer |
| Prices | Inside `prices{}` object | Top-level strings |
| Variation images | `images[]` array (max 1) | `image` singular object |
| Data completeness | Limited | Full — same as WooCommerce admin dashboard |

The admin API is strictly better. Real stock quantities and complete product data make every downstream step more reliable.

### Variation image shape
The admin API returns variation images as a **singular `image` object**, not an array. Reshape converts it to `images[]` for consistency with the product schema.

### Slugs don't derive from names
WooCommerce slugs diverge from product/category names over time: apostrophes get dropped, names get truncated, manual slug edits happen. Never try to regenerate a slug from a name — treat slugs as opaque strings and just validate they're non-empty.

### Attribute options can be empty on the product
A variable product's attribute list shows what attributes exist, but the `options[]` array can be empty at the product level. The actual option values live on each variation (`option` field, the term name). `create_items.py` expands options from variation data, so empty options at the product level are fine — don't flag them.

### `on_sale` is unreliable on variable products
The WooCommerce `on_sale` boolean on a variable product is an aggregate that doesn't always reflect what's happening at the variation level. For variable products, `sale` is best determined by looking at variation prices directly. For simple products, `on_sale` + `sale_price` are consistent and reliable.

---

## Data shape learnings

### Variable product stock should be null
WooCommerce stores an aggregate `stock_quantity` on variable products that rolls up from variations, but it's often stale or misleading. The real source of truth is variation-level stock. Reshape sets `stock=null` for all variable products — the DB should reflect that variations own stock for variable products.

### Prices in cents as integers
All prices are stored as integers in cents (e.g. $12.99 → 1299). This avoids floating-point rounding errors entirely. `round(float(price_str) * 100)` converts cleanly.

### Description images extracted separately
Product descriptions are HTML and often contain `<img>` tags embedded in the copy. Reshape strips all HTML from the description text but extracts those image URLs into a separate `descriptionImages` array as `{src, name}` objects. The DB stores them separately from product images.

### Image deduplication at variation level
Most variations share the same image as the first variation for that product (WooCommerce just repeats the same URL). `create_items.py` sets `images=[]` on any variation whose image URL matches the first variation's image — deduplication reduces redundant image downloads and uploads. Only variations with a truly unique image keep their `images` array populated.

### Attribute name cleaning
WooCommerce attribute names are messy — filter-related suffixes and prefixes get added for the storefront UI but are meaningless in the DB:
- Strip `for Filter` suffix (e.g. `Frame Styles for Filter` → `Frame Styles`)
- Strip `Choose ` prefix (e.g. `Choose Color` → `Color`)
- Strip `Filter by ` prefix (e.g. `Filter by Lens Type` → `Lens Type`)
- Strip parentheticals like `(REQUIRED)`

Some names also need brand-specific hardcoded overrides (e.g. `Cleaner Spray Bottle QTY` → `Quantity`, `Qty` → `Quantity`) because they're too unusual for generic rules to handle cleanly.

### Attribute name collisions
After cleaning, two different raw attribute names can map to the same cleaned name within one product (e.g. `Lens Type for Filter` and `Filter by Lens Type` both become `Lens Type`). This has to be caught before reshape — validate scripts simulate the cleaning function and flag any product where a collision would occur. These require a manual fix in WooCommerce (remove the duplicate attribute).

---

## Multi-brand architecture

### One env var, one config dict
`BRAND` in `.env` selects the active brand. `BRAND_CONFIG` in `config/__init__.py` maps brand keys to slug and credential env var names. Every script calls `load_brand()` at startup — no hardcoded brand names anywhere in the pipeline scripts.

Adding a new brand: add one entry to `BRAND_CONFIG`, add the credential env vars to `.env`, write a brand-specific categorize script.

### Brand-specific attribute overrides in reshape scripts
`ATTR_OVERRIDES` is a brand-keyed dict inside `reshape_products.py` and `reshape_variations.py` (not in config). Brand-specific data transformation logic belongs in the transform scripts, not the shared config. When a new brand needs custom attribute name mappings, add a new key to the dict.

### Categorize is always brand-specific
Category mapping is entirely subjective — what counts as a leaf category, how to normalize category names, and what to do with uncategorized products varies by brand. Each brand gets its own `{brand}_categorize_items.py`. The output format is standardized (`[[{level, name, slug}]]` arrays) but the mapping logic is not shared.

---

## What causes products/variations to get flagged

### Products
| Issue | Fix |
|-------|-----|
| No SKU | Add SKU in WooCommerce |
| SKU contains spaces | Clean SKU in WooCommerce |
| Duplicate SKU | Resolve in WooCommerce |
| No images | Add at least one product image |
| Status not `publish` (draft, private) | Publish the product |
| Attribute name collision after cleaning | Remove the duplicate attribute in WooCommerce |
| type=variable but no variation IDs | Add variations or change type |

### Variations
| Issue | Fix |
|-------|-----|
| SKU contains spaces | Clean SKU in WooCommerce |
| No price | Set regular_price on the variation |
| Invalid attribute name or option | Fix in WooCommerce |

### Create items
A product gets flagged at create-items if one or more of its variation IDs was flagged at validate-variations — the variation is absent from the reshape output, leaving a dangling ID. Fix the variation upstream, re-fetch, re-run from that stage forward.

---

## QA mindset

- **Audit tells you what the data looks like.** Run it to sanity-check distributions, catch outliers, and understand coverage before moving on.
- **Verify tells you whether the transformation was correct.** It's deterministic — PASSED or FAILED with specific errors.
- **Always run both after every stage.** Audit catches things verify doesn't (unusual values, low coverage) and verify catches things audit doesn't (silent wrong transformations).
- **Verify scripts must mirror reshape logic exactly.** If reshape gets a new cleaning rule, verify needs the same rule or it produces false failures. Keep them in sync.

---

## ProSport — final numbers

| Table | Rows |
|-------|------|
| products | 166 |
| variations | 1,713 |
| categories | 12 (4 roots, 8 leaves) |
| product_categories | 197 |
| product_images | 565 |
| variation_images | 36 |
| description_images | 143 |

3 products pending fix in WooCommerce (SKU spaces + missing price): 2145, 2211, 1208.

---

## Monster — current numbers (through create-items)

| Stage | Count |
|-------|-------|
| Products fetched | 277 |
| Products validated | 267 (10 flagged) |
| Variations validated | 2,673 (2 flagged) |
| Items created | 265 (2 flagged) |
| Variations embedded | 2,646 |

12 items to fix in WooCommerce before they can be imported.
