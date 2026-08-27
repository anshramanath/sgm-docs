# What I Built and Learnt — Bikershades Migration (May 31, 2026)
_Last updated: 2026-05-31_

---

## What I Built

### The Migration Pipeline (7 steps, end-to-end)

A fully automated, crash-safe pipeline that takes raw WooCommerce API data and lands it cleanly in a new Supabase/Postgres database.

| Step | Script | What it does |
|------|--------|--------------|
| 1 | `fetch_products.py` | Paginates WooCommerce Store API, saves 647 products |
| 2 | `fetch_variations.py` | Fetches 6,064 variations (crash-safe, resumes per-variation) |
| 3 | `reshape_items.py` | Normalises schema, validates fields, buckets problems |
| 4 | `recategorize_items.py` | Rebuilds category tree: drops, merges, collapses, strips non-leaves |
| 5 | `download_images.py` | Downloads all images locally (crash-safe via `url_map.json`) |
| 6 | `upload_images.py` | Uploads to Supabase Storage (crash-safe via `upload_map.json`) |
| 7 | `import_items.py` | Inserts everything into Supabase DB in FK order |

### The QA System (10 scripts)

A full audit + verify layer for every pipeline stage. Each pair catches different things:
- **audit** — validates the output file itself (field presence, structure, invariants)
- **verify** — cross-references two stages to prove no data was lost or corrupted

| Stage | Audit | Verify |
|-------|-------|--------|
| reshape | `audit_shaped_items.py` | `verify_shaped_items.py` |
| recategorize | `audit_categorized_items.py` | `verify_categorized_items.py` |
| download | `audit_downloaded_images.py` | `verify_downloaded_images.py` |
| upload | `audit_uploaded_images.py` | `verify_uploaded_images.py` |
| import | `audit_import.py` | `verify_import.py` |

### The Database Schema

Multi-brand Supabase/Postgres schema designed from scratch:

- `brands` → `categories` (self-referential tree) → `products` → `variations`
- Junction table `product_categories` for many-to-many product↔category
- Separate image tables: `product_images`, `variation_images`, `description_images`
- `admins` table gates admin access (no public Supabase signup)
- RLS on all tables; frontend never hits Supabase directly — goes through Next.js with `service_role` key

### The Image Pipeline

A two-phase deduplication system:
1. **Download phase:** `url_map.json` maps each unique bikershades URL → local path. Same image referenced across 50 products is downloaded exactly once.
2. **Upload phase:** `upload_map.json` maps local path → Supabase storage URL.
3. **Import phase:** resolves `bikershades URL → url_map → local path → upload_map → storage URL`. If either lookup fails, the image is silently dropped — no broken URLs ever enter the DB.

### Documentation

- `PROGRESS.md` — full decision log
- `CLAUDE.md` — auto-loaded context for Claude Code sessions
- `docs/shaped-items/` — audit, verify, attributes, categories
- `docs/categorized-items/` — audit, verify, category tree, problem reviews
- `reshaped-problems/` — per-bucket triage notes for all flagged products

---

## What I Learnt

### WooCommerce API

- The Store API (`/wp-json/wc/store/products`) returns parent products only — variations are excluded from pagination by design. You need a second pass per product to get full variation data.
- Variation attributes come in two forms: a stub in the products endpoint (`attributes: [{name, value}]`) and a raw string in the variation object (`variation: "Choose Color: Black"`). The stub is more reliable — the string was empty on 38 variations.
- `stock_status` is null on every product. The real stock signal is `is_in_stock` (bool).
- Prices are **strings in cents**: `"1999"` = $19.99.
- `on_sale` from WooCommerce is not reliable — some products marked `on_sale: true` have `sale_price == regular_price`. Always re-derive from the price comparison.
- `formatted_weight` and `formatted_dimensions` are the only source of units. Last word of the string (e.g. `"6 oz"` → `"oz"`). Can be `"N/A"` — must validate.

### Data Quality in the Wild

Raw WooCommerce data is messier than expected:
- **Duplicate slugs under different formats:** `1-00` and `1.00` both stored for the same power diopter.
- **Capitalisation inconsistencies:** `black` and `Black` coexist as separate terms.
- **Attribute drift:** variation slugs that don't match any term in the parent product's attribute list — a sign of WooCommerce data that was edited inconsistently over time.
- **Empty attribute arrays:** 5 products where variation option metadata is completely missing from the API — not recoverable programmatically.
- **Orphaned categories:** WooCommerce has structural "see all" categories (`see-all-brands`, `see-all-rx-frames`) that exist purely as nav UI. They need to be dropped from the real category tree.
- **Ghost nodes:** some categories have items assigned to them in WooCommerce but the node is just a structural placeholder — no products should land directly there.

### Schema Design Decisions

**Flatten vs normalise:** variation images went through two iterations — first flattened to `image_src`/`image_name` columns on the variation row (since WooCommerce has 1 per variation), then moved to a proper `variation_images` table. The table approach is more future-proof even if it's slightly more complex now.

**Nullable vs not null:** learned to be careful about `NOT NULL DEFAULT '...'` — if you pass an explicit `NULL` in an INSERT, Postgres does not apply the DEFAULT; the NOT NULL constraint fires instead. So nullable columns must be declared `text` (no constraint), not `text not null default 'oz'`.

**Price model:**
- Variable products: `min_price_cents`/`max_price_cents` derived from variation **regular** prices (not effective/sale prices). The product row shows the regular price range. `sale_price_cents` is always null on the product.
- Simple products: `min = max = regular_price`. `sale_price_cents` is null when not on sale.
- Variations: `sale_price_cents` is null when `sale=false`, set (< `regular_price_cents`) when `sale=true`.
- The `sale` bool is always authoritative — never derive it from price comparison at read time; compute and store it at write time.

**Stock as int not bool:** `stock int` (0 or 1 from `is_in_stock`) rather than a boolean. Owner can later update to real quantities; frontend treats `stock > 0` as in stock. No schema change needed when real inventory tracking is added.

**Categories as a self-referential tree:** `parent_id uuid references categories(id)` — null = root. Enables the owner to create, move, and rename categories through an admin UI without any code changes. Hierarchy is reconstructed at query time from parent_id chains. Import builds the tree lazily as products are processed.

### Crash-Safe Scripting

The fetch/download/upload scripts can all be interrupted and resumed with zero data loss:
- After every unit of work (one variation fetched, one image downloaded, one image uploaded), write the checkpoint file.
- On restart, load the checkpoint and skip everything already in it.
- This means the checkpoint IS the output — no separate "done" state to manage.

### Image Deduplication

The insight: deduplicate by URL, not by file. The same `https://bikershades.com/...jpg` is referenced by many products/variations. Keying `url_map.json` by URL means it's downloaded once regardless of how many places reference it. The folder structure under `images/local/` is cosmetic — the map is the source of truth.

### Category Tree Restructuring

WooCommerce categories are a mess of:
- Structural "see all" nodes that exist purely as nav elements
- Placeholder parents with no real products
- Leaf nodes that only have 1 product (not useful as a category)
- Near-duplicate slugs (`mirrored` and `mirror`, `small-medium` and `small-medium-shop-by-head-size`)

The recategorization script applies rules in order — drops first, then merges, then collapses single-item leaves iteratively (collapsing can create new single-item leaves, so it loops until stable). Products are stripped to leaf-only paths so no product ever sits at both a branch node and a leaf.

### Bucketing vs Silently Passing Through

A key design call made repeatedly: when data is bad, do you bucket the product or fix/null the field and let it through?

The rule that emerged:
- **Bucket** when the field is required and can't be safely derived (weight, SKU, price, images)
- **Silently null** when the field is optional and the bad data is recoverable as "absent" (partial dimensions, bad dimension unit)

Dimensions are optional for shipping. Weight is required. So: bad/partial dimensions → null them and pass through. Bad/missing weight → bucket to `invalid_weight_items`.

### Attribute Cleanup

After reshape, some product attribute terms had no matching variation — clicking that option on the frontend would find zero results. The fix: after merging variation data, strip any product attribute term not referenced by at least one variation. This ensures every color swatch on a product page leads somewhere.

### QA Design

Writing checks that catch real bugs vs checks that are trivially true:
- **Cross-reference two stages** rather than just validating structure — e.g. verify that `min_price_cents` in shaped output actually equals the minimum of that product's variation `regularPriceCents` values.
- **Check invariants in both directions** — if `sale=true` then `salePriceCents` must be non-null AND if `salePriceCents` is non-null then `sale` must be true.
- **Separate critical from informational** — an empty `summary` is expected for 32% of products; a null `sku` is never OK. Mixing these in a single error output makes the critical issues invisible.
