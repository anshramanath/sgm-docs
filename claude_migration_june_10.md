# Learnings (Jun 10, 2026)

Everything learned and built during the WooCommerce → Supabase migration.

---

## Architecture decisions

### Multi-brand with a single pipeline
All three brands (proSPORT Sunglasses, Sunglass Monster, BikerShades) run through the same pipeline scripts. Brand is injected at runtime via `BRAND=<key>`. A single `config/__init__.py` holds all brand-specific values. This keeps scripts generic and avoids duplication.

### Two slug fields per brand
- `slug` — pipeline key, data directory name, used internally (`prosport`, `monster`, `bikershades`)
- `brand_slug` — the actual DB slug and Supabase bucket name, derived from the brand display name (`prosport-sunglasses`, `sunglass-monster`, `bikershades`)

These must never be conflated. Early mistake was using `slug` for the bucket name, which gave buckets the wrong names.

### Per-brand import, not truncate
Import deletes only `WHERE slug = brand_slug`, which cascades via FK constraints to all that brand's data. Using `TRUNCATE CASCADE` wiped all brands — discovered and fixed early.

### No fallbacks in import
All field accesses in `import_items.py` are direct (`p["field"]`). If a key is missing, the script crashes. This is intentional — a crash means a validation gap upstream that needs fixing, not a case to handle at import time.

### Deferred fields
`description`, `summary`, and `categories` are left null/empty through the reshape and early validation stages. They're filled by the MCP content generation step and categorize scripts respectively. `validate_items.py` skips them; `validate_items_final.py` checks them.

---

## Pipeline learnings

### Two validation scripts, different purposes
- `validate_items.py` — early structural check, runs before content generation. Only checks non-deferred fields. Used as a gate before MCP content generation.
- `validate_items_final.py` — thorough pre-import check. Runs after all fields are populated. Checks everything: slug format, duplicate slugs across catalogue, image uniqueness, variation attribute combos, price consistency, category structure.

### Checks accumulate per entity
A product collects all failures before being flagged. One failure is enough to flag it. This means you see the full picture of what's wrong with each product rather than stopping at the first issue.

### Duplicate variation attribute combinations
Two variations with identical attribute combos (e.g. both `color=Black, power=+1.50`) create ambiguous storefront routing — the frontend can't tell which to show. These are flagged by both validators. Root cause is WC data entry error.

### WooCommerce "Any" attribute
When a variation's attribute is set to "Any" in WooCommerce, the API omits that attribute entirely from variation data. This means some variations appear to be missing attributes when they're not — they just inherit them from the product level. Not fixable in the pipeline without WC data correction.

### Variable vs simple product rules
- Variable: `sku` is null at product level; SKU/Veeqo checks apply only at variation level
- Simple: `sku` must be present; `variations` must be empty
- Bad variations (not in Veeqo, not an object) are dropped silently — a variable product needs ≥2 surviving variations

### Veeqo always wins
Veeqo CSV data takes priority over WooCommerce for both price and stock. `0` is a valid Veeqo value — not treated as "no data". If a SKU appears multiple times in Veeqo: highest price wins, lowest stock wins.

### Variable product fields are derived from children
`minPriceCents`, `maxPriceCents`, `inStock`, and `attributes` on a variable product are all computed from its surviving variations after Veeqo enrichment — not taken from WooCommerce product-level data.

---

## Image pipeline learnings

### Image uniqueness is URL-based
An image is unique if its original WC URL is unique. Multiple items/variations can reference the same URL — they all resolve to the same Supabase public URL at import time. The pipeline downloads each unique URL exactly once.

### Two image naming conventions
1. **`name` field** (stored in DB, display use): URL basename stem with `-` and `_` replaced by spaces. e.g. `2018/01/Black-Angle-3.jpg` → `Black Angle 3`. Done by `normalize_image_names.py`.
2. **Downloaded filename** (on disk, in Supabase): everything after `/uploads/` or `/gallery/` with `/` replaced by `_`. e.g. `uploads/2018/01/Black-Angle-3.jpg` → `2018_01_Black-Angle-3.jpg`. Guarantees uniqueness across different year/month upload directories.

### Image checks in validate_items_final
- All product images within an item must have unique `src` URLs
- No variation image may duplicate a product-level image `src` (variations inherit parent images implicitly)
- `sortOrder` values must be unique within each image array

### Download failures
- **404**: image is gone from the WC server — not recoverable
- **503**: server temporarily unavailable — retryable and usually recovers on the next run
- All failed image downloads are variation-level images; every item has at least one product image downloaded (verified as a post-download check)

### Crash-safe download and upload
Both `download_items.py` and `upload_items.py` write progress to `download_map.json` / `upload_map.json` after every successful operation. Re-running picks up exactly where it left off.

### Supabase storage bucket naming
Buckets are named after `brand_slug`, not `slug`. One bucket per brand. All images for a brand go into a flat structure within their bucket (no subdirectories).

---

## Database learnings

### camelCase pipeline → snake_case DB
All pipeline JSON uses camelCase field names. The import script maps these to snake_case DB column names at insert time. Never change field names mid-pipeline — the mapping only happens at the final step.

### Category upsert pattern
Categories are upserted lazily as products are processed. The key is `(parent_id, slug)` — two categories with the same slug under different parents are distinct nodes. The `category_id_map` dict accumulates across all products in a single import run.

### Per-brand cascade delete
The `brands` table has FK relationships to `products`, `categories`, etc. `DELETE FROM brands WHERE slug = %s` cascades cleanly to all related rows. Schema is in `final-migration/db/001_initial_schema.sql`.

---

## Infrastructure learnings

### Supabase direct connection and IPv6
The direct Postgres connection (`db.xxx.supabase.co`) uses IPv6 on newer Supabase projects. Corporate/office networks often block IPv6, causing DNS resolution to fail entirely. Fix: use the session pooler URL (`aws-0-region.pooler.supabase.com`) from Supabase dashboard → Settings → Database → Connection string.

### MCP content generation
AI-generated descriptions and summaries are produced via a local MCP server (`mcp/server.py`) and Claude Desktop. The server exposes three tools: `get_next_product`, `save_content`, `get_progress`. Validation is enforced server-side — Claude Desktop must retry on `VALIDATION FAILED`. Known issue: duplicate product slugs cause infinite loops; fix manually by copying saved content in the JSON.

---

## What was built (final-migration)

| Script | Purpose |
|---|---|
| `fetch_products.py` | WC API fetch with simple product validation |
| `create_items.py` | Embed variations into products |
| `slim_items.py` | Strip to migration-relevant fields |
| `reshape_items_1.py` | Transform to DB shape, WC values |
| `reshape_items_2.py` | Veeqo enrichment + variable product derivation |
| `validate_items.py` | Early structural validation |
| `remove_items.py` | Filter manually flagged items |
| `merge_attributes.py` | Canonicalize WC attribute names |
| `dedup_images.py` | Remove duplicate and redundant images |
| `generate_content.py` | Initialize content generation stage |
| `mcp/server.py` | AI content generation via Claude Desktop |
| `{brand}_categorize_items.py` | Assign category paths |
| `normalize_image_names.py` | Normalize image name fields from URLs |
| `validate_items_final.py` | Thorough pre-import validation |
| `download_items.py` | Download all images locally, crash-safe |
| `upload_items.py` | Upload images to Supabase storage, crash-safe |
| `import_items.py` | Insert everything into Supabase DB, per-brand |
