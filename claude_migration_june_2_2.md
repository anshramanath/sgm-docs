# Sunglass Monster Migration — Learnings (June 2, 2026)
_Built 2026-05-xx → 2026-06-02 | ProSport run complete_

Everything learned from designing and running a WooCommerce → Supabase migration pipeline from scratch. Written so future-you (or a second brand) can skip the trial-and-error.

---

## What was built

A staged, auditable data migration pipeline:

1. **Fetch** raw product and variation data from the WooCommerce admin REST API
2. **Slim** to only the fields that matter
3. **Validate** — hard fail on bad data, write flagged items to a separate file
4. **Reshape** — normalize to the import schema (strip HTML, prices to cents, clean names)
5. **Create items** — embed validated variations into their parent products, compute min/max prices, dedup images
6. **Categorize** — replace WooCommerce category tags with a clean structured tree
7. **Download** all unique images locally (crash-safe)
8. **Upload** to Supabase Storage (crash-safe)
9. **Import** into Supabase/Postgres via psycopg2

Each stage writes to `data/{slug}/{stage}/` and never mutates earlier output. You can re-run any stage forward without re-fetching from the API.

---

## The APIs

### WooCommerce admin REST API vs store API

BikerShades used the unauthenticated store API (`/wp-json/wc/store/v1/`). ProSport uses the authenticated admin API (`/wp-json/wc/v3/`). This matters:

| | Store API | Admin API |
|---|---|---|
| Auth | None | HTTP Basic (consumer key + secret) |
| Stock | `is_in_stock` bool only | Real `stock_quantity` integer |
| Prices | Inside `prices{}` object | Top-level strings |
| Variation images | `images[]` array | `image` singular object |
| Data completeness | Lossy | Full |

Always use the admin API for migrations. The store API is designed for storefronts, not data access.

### Pagination

The admin API paginates at 100 items per page by default. `fetch_products.py` loops until an empty page is returned. Variations are fetched per-product in a nested loop — each product's variations need their own paginated fetch.

---

## WooCommerce data quirks

Things that aren't obvious from the docs and will bite you:

**`on_sale` on variable products is always True when any variation is on sale.**
154/171 ProSport products had `on_sale=True` with no parent `sale_price`. Ignore `on_sale` on variable products entirely — only use it on simple products.

**`stock_quantity` is only valid when `manage_stock=True`.**
For parent products, only 19/171 had stock management on. For variations, all 1758 had it on — but you can't assume that carries across brands. Always check `manage_stock` before trusting `stock_quantity`. Use `stock_status` (`instock` / `outofstock`) as the primary signal.

**Parent product SKUs on variable products are meaningless.**
The real SKUs live on the variations. Null them at reshape.

**`dimensions` object always exists but all fields are always empty strings.**
Never `{}`, never null — always `{"length": "", "width": "", "height": ""}`. You have to handle it even though it carries no data. Coerce all three to float or null at reshape.

**Every product is tagged `Uncategorized` alongside its real categories.**
Strip it at categorize. It's a WooCommerce default that applies to every product before the store owner sets real categories.

**`type` field is unreliable.**
Use `len(variations) > 0` as the source of truth for whether a product is variable. The `type` field occasionally disagrees with the actual variation data.

**Variation attribute `option` is the term name, not the slug.**
Parent products have `attributes[].options[]` (array of strings). Variations have `attributes[].option` (singular string). Both are human-readable names like `"Matte Black"`, not slugs. Don't slug-ify them.

---

## The data model

### Variations go inside products, not alongside them

WooCommerce stores products and variations as separate tables linked by `parent_id`. For the import target, variations are embedded inside their parent product object. This is the create-items step.

The process: reshape-variations produces `[{product_id, variations: [...]}]`. create-items walks each product's `variations[]` array (which at that point is still just IDs), finds the matching variation object by ID, and swaps it in-place. After this step every product carries its full variation data.

### Image deduplication

Variation images are almost always a subset of the parent product gallery. After embedding variations into products, any variation image whose `src` matches a parent gallery image is removed. For ProSport: 1677/1713 variations ended up with `images: []` after dedup — only 36 carried a truly unique image.

Dedup is always `src`-only. Never use `name` for uniqueness checks — `name` from WooCommerce is unreliable and not stable.

### The two-map image chain

Images go through two maps:

```
original WooCommerce URL
  → download_map.json → local file path
  → upload_map.json   → Supabase storage URL
```

Both maps key on the thing before them. `download_map` keys on original URL, `upload_map` keys on local path. At import time, resolving an image means two lookups. If either fails, the image is silently dropped.

Crash safety: both maps are appended to disk after every single success. A killed process resumes exactly where it left off — already-downloaded or already-uploaded files are skipped.

### Image filenames

Download step derives filenames from the URL path — everything after `/uploads/` with `/` replaced by `_`:

```
.../wp-content/uploads/2020/11/Cabriolet-Gloss-Black-Front.jpg
→ 2020_11_Cabriolet-Gloss-Black-Front.jpg
```

The year/month prefix prevents collisions between files with the same basename uploaded in different months. Upload to Supabase uses the same flat filename inside the brand bucket.

### description images

Product descriptions are HTML blobs. At reshape, `<img>` tags are extracted into `descriptionImages: [{src, name}]` before stripping HTML. `name` = just the filename at the end of the URL path (`rsplit('/', 1)[-1]`), not the full `uploads/year/month/...` path. Uniqueness isn't needed here — it's for display only.

---

## Schema design

### Normalized categories

Categories are stored as a self-referencing table:

```sql
categories (
  id uuid primary key,
  parent_id uuid references categories(id),
  name text not null,
  slug text not null,
  unique(parent_id, slug)
)
```

The pipeline produces `categories: [[{level, name, slug}, ...], ...]` — an array of path arrays. Each path is walked root-to-leaf at import time via `ensure_category(path)`, which lazily inserts missing nodes and caches `(parent_id, slug) → uuid`. Products link to their leaf category UUIDs via `product_categories`.

### Variation attributes as jsonb

Storing `[{name, option}]` as jsonb was the right call. The alternative — a relational `attributes` table — would require joining every time you read variations and adds complexity for no real benefit at this scale. jsonb queries are fast enough and the structure is simple.

Don't reformat the data for storage. Pass the array as-is from reshape: `Json(v.get("attributes", []))`.

### Price in cents, not floats

All prices stored as `int` (cents). WooCommerce sends prices as strings (`"19.99"`). Convert at reshape: `round(float(price) * 100)`. Never store floats — rounding errors accumulate.

`regularPriceCents` and `salePriceCents` on variations. `minPriceCents`, `maxPriceCents`, `salePriceCents` on products. Product-level min/max is computed from variation `regularPriceCents` at create-items — uses the stable "was" price, not the current selling price, so the range doesn't change when sales start/stop.

### Field naming convention

JSON field names in reshape output match DB column names with `_` removed and camelCased:
- `old_url` → `oldUrl`
- `total_sales` → `totalSales`
- `min_price_cents` → `minPriceCents`
- `summary` (not `short_description`) → `summary`

This means verify scripts can compare JSON fields to DB columns by column name without a mapping table.

### `total_sales` — treat zero as unknown

80/171 ProSport products had `total_sales > 0`. The other 91 had 0, but WooCommerce's count can reset across migrations and doesn't always reflect real history. Treat 0 as "unknown", not "confirmed zero sales". Store as `int not null default 0` — null from WooCommerce becomes 0 at reshape (`p.get('total_sales') or 0`).

---

## Category design

### Brand-specific categorize script

Category normalization is always brand-specific — different WooCommerce tag names, different desired final tree, different decisions about homeless products. Each brand gets its own script (`prosport_categorize_items.py`) with hardcoded `RENAME_MAP`, `TREE`, and `HARDCODED` assignments.

### Only leaves carry products, parents are structural

Products are always assigned to leaf categories. Parent categories exist only to group leaves. This keeps queries simple: get all products in a category = find all products whose leaf category_id matches.

### The path array structure

`categories: [[{level, name, slug}]]` — outer array = list of paths, inner array = ordered nodes from root to leaf. A flat list would lose the parent→child relationship for products appearing in multiple branches. The nested structure makes it unambiguous.

### Naming rules

- Title case — every word capitalised except `and`
- No `&` in names — use `and` so the round-trip `slug → name → slug` is clean
- Slug = `name.lower().replace(' ', '-')`
- Verify: `slugify(name) == slug` and `namify(slug) == name` must both hold

---

## Validation design

### Fail early, fail completely

Each validate script runs every check against every product/variation and accumulates all failures. A flagged item goes to `flagged.json` with its full error list. This means you see the whole picture in one pass, not just the first error per item.

### Flagged = excluded, not blocked

Flagged items are written to `flagged.json` and excluded from the output. The pipeline continues with valid items. You don't need to fix everything before you can run — you can import 166 products now and add the 3 flagged ones after fixing WooCommerce.

### ProSport flags

| Item | Problem | Fix |
|---|---|---|
| Product 4019 | Draft status, no slug — junk copy | Drop |
| Product 3139 | No SKU | Assign SKU in WooCommerce |
| 17 variations (products 2145, 2211) | Errant space in SKU before `-MFS`/`-MTO` suffix | Fix SKUs in WooCommerce |
| Variation 1355 (product 1208) | Price field empty | Assign price in WooCommerce |

---

## QA pattern

Every stage has two scripts:

**`audit_{stage}.py`** — prints counts, distributions, and potential issues. Non-blocking. Run after every stage to understand the data.

**`verify_{stage}.py`** — strict checks that must all pass before moving forward. Exits with PASSED or FAILED + error list.

The verify scripts compare the stage output back to the previous stage's output, ensuring nothing was silently dropped or transformed incorrectly. At import, verify compares DB row counts and field values directly to the JSON source.

---

## Multi-brand setup

### One BRAND_CONFIG per script

Every pipeline, analysis, and QA script has:

```python
BRAND_CONFIG = {
    "prosport": {"slug": "prosport", "brand_name": "proSPORT Sunglasses"},
}
brand_name = os.getenv("BRAND")
config = BRAND_CONFIG.get(brand_name)
slug = config["slug"]
```

`BRAND=prosport` in `.env` selects the active brand. All output writes to `data/{slug}/`. Switching brands is one env var change.

### Supabase bucket = slug

Each brand gets its own Supabase Storage bucket named after the slug (`prosport`). This isolates assets cleanly — no cross-contamination when adding a second brand.

---

## Run results — ProSport (2026-06-02)

| Stage | In | Out | Flagged |
|---|---|---|---|
| fetch_products | — | 171 | — |
| validate_products | 171 | 169 | 2 (draft + no SKU) |
| fetch_variations | 169 products | 1758 | — |
| validate_variations | 1758 | 1740 | 18 (SKU spaces + no price) |
| reshape_products | 169 | 169 | — |
| reshape_variations | 1740 | 1740 | — |
| create_items | 169 + 1740 | 166 items, 1713 variations | 3 (unreplaced IDs from flagged variations) |
| prosport_categorize_items | 166 | 166, 197 leaf assignments | — |
| download_items | — | 411 images | 0 failed |
| upload_items | 411 | 411 → Supabase | 0 failed |
| import_items | 166 items | 166 products, 1713 variations, 12 categories in DB | — |

All verify scripts: **PASSED**

---

## What to do for the next brand

1. Get admin API credentials (consumer key + secret), add to `.env` as `{BRAND}_BASE_URL`, `{BRAND}_CONSUMER_KEY`, `{BRAND}_CONSUMER_SECRET`
2. Add the brand to `BRAND_CONFIG` in every script (or build a shared config import)
3. Set `BRAND={slug}` in `.env`
4. Run fetch → slim → validate — read the analysis outputs carefully before going further
5. Write a `{slug}_categorize_items.py` with the brand's actual category tree
6. Run the rest of the pipeline
7. Create `docs/run-1/` for this brand's first run notes
