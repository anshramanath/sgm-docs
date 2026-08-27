# Migration Learnings & Build Notes (June 14, 2026)

A full account of what was built, what broke, and what was learned migrating three WooCommerce catalogues into a Supabase-backed multi-brand platform.

---

## What Was Built

A 17-step Python pipeline that takes raw WooCommerce API data and produces a clean, validated, fully imported Supabase database across three brands.

### Brands

| Brand key | DB slug | Display name | Products |
|---|---|---|---|
| `prosport` | `prosport-sunglasses` | proSPORT Sunglasses | 150 |
| `monster` | `sunglass-monster` | Sunglass Monster | 256 |
| `bikershades` | `bikershades` | BikerShades | 464 |

### Pipeline Overview

```
WooCommerce API
  → fetch_products         filter invalid simple products
  → create_items           embed variations into products
  → slim_items             strip unused WC fields
  → reshape_items_1        shape + WC prices → cents
  → reshape_items_2        Veeqo enrichment + variable product derivation
  → validate_items         non-deferred field checks
  → remove_items           hardcoded exclusion list
  → merge_attributes       normalize WC attribute names
  → dedup_images           remove duplicate + inherited variation images
  → generate_content       null out descriptions for AI regeneration
  → MCP server             Claude Desktop writes descriptions + summaries
  → categorize_items       brand-specific hardcoded category trees
  → normalize_image_names  derive image name from URL stem
  → validate_items_final   thorough pre-import validation
  → download_items         download all images to disk
  → upload_items           upload to Supabase storage buckets
  → import_items           write to Supabase Postgres
```

---

## DB Schema Design

### Key decisions

**Brands as the root.** Every table traces back to `brands` via `brand_id`. Deleting a brand cascades and wipes all its data cleanly.

**Two slug fields per brand:**
- `slug` — pipeline key, used in data directories and storage bucket paths (`prosport`, `monster`, `bikershades`)
- `brand_slug` — DB slug, Supabase bucket name (`prosport-sunglasses`, `sunglass-monster`, `bikershades`)

Never conflate them. Early bug: upload was using `slug` instead of `brand_slug` for bucket naming so images went to the wrong bucket.

**Description images join table.** The same image URL can appear in multiple products' descriptions. Rather than duplicating rows, `description_images` is a brand-scoped image bank (`brand_id, src` unique) and `product_description_images` is a join table linking products to images. This also enables future UI: when creating a product, a user can pick from existing brand images rather than re-uploading.

```sql
description_images (id, brand_id, src, name)      -- unique per brand+src
product_description_images (product_id, image_id)  -- join table
```

`brand_id` is stored directly on `description_images` (rather than being derived via joins through products) for query performance — it could theoretically be derived, but the direct column is faster.

**No sort_order on description images.** Order is implicit in the product's HTML description. Tracking it separately would duplicate information and go stale.

**ON DELETE CASCADE throughout.** All FKs pointing at `brands` and `products` have cascade so a future `DELETE FROM brands WHERE slug = X` wipes everything for that brand cleanly. The import script also does explicit ordered deletes as a belt-and-suspenders measure (see: the partial import bug).

**camelCase in pipeline, snake_case in DB.** All pipeline JSON uses camelCase. The import script handles the mapping explicitly — no automatic conversion.

**No DB-level unique constraint on product name/slug within a brand.** The validator enforces slug uniqueness at pipeline time. Name/slug uniqueness within a brand is enforced at the application layer on the frontend instead.

---

## Key Technical Learnings

### 503 vs 404 on image downloads

503 = server temporarily unavailable (retryable). 404 = genuinely gone.

Early downloads returned 503s for some bikershades and monster images. The server was returning 503 under batch load but the images didn't actually exist. Retrying one-at-a-time (not batched) revealed they were all 404s. Confirmed by retrying multiple times until stable.

Final tally of genuinely missing images: 1 monster product image, 78 bikershades images (variation + description). All confirmed 404.

**Rule of thumb:** always retry 503s individually before treating them as permanent failures.

### IPv6 DNS failure on corporate networks

Direct Supabase connection (`db.xxx.supabase.co`) resolves to an IPv6 address. Corporate/office networks often block IPv6 DNS resolution, causing:

```
psycopg2.OperationalError: could not translate host name "db.xxx.supabase.co"
```

Fix: use the Supabase **session pooler URL** instead. It resolves over IPv4 and works everywhere.

### The partial import bug

If an import run crashes mid-way, the last committed checkpoint is preserved (commits every 50 products). The brand exists in the DB in a partial state.

On the next re-import, `DELETE FROM brands WHERE slug = X` fails with a FK violation because `categories` references `brands(id)` and the live DB FKs didn't have `ON DELETE CASCADE` at the time.

The fix: delete in explicit reverse dependency order before re-inserting:

```
variation_images
→ product_description_images
→ product_categories
→ product_images
→ variations
→ products
→ description_images
→ categories
→ brand
```

Root cause: the delete was always a no-op before (brand didn't exist yet on first import of each brand). The bikershades crash was the first time a brand existed in the DB when we tried to re-import it.

### Upload map as resume mechanism

`upload_map.json` maps local file path → Supabase public URL. Upload skips files already in the map. When storage buckets were wiped externally, the map still had the old entries so re-running upload did nothing (all "skipped (resume)").

Fix: delete the upload maps before re-uploading after a bucket wipe.

### Upsert pattern for description images

```python
INSERT INTO description_images (brand_id, src, name)
VALUES (%s, %s, %s)
ON CONFLICT (brand_id, src) DO UPDATE SET name = EXCLUDED.name
RETURNING id;
```

This always returns the id whether it inserted or found an existing row. Combined with an in-memory `desc_image_id_map` cache (src → id), each unique image is only round-tripped to the DB once per import run.

### Image filename uniqueness

Downloaded image filenames use everything after `/uploads/` or `/gallery/` in the URL path, with `/` replaced by `_`. This guarantees uniqueness within a brand without relying on date prefixes or other metadata that can be ambiguous.

Example: `https://example.com/wp-content/uploads/2018/01/Black-Angle.jpg` → `2018_01_Black-Angle.jpg`

### Variable product derivation

WooCommerce stores variable products and their variations separately. The pipeline:
1. Fetches each variable product's variations from the WC API
2. Drops bad variations (not in Veeqo, missing attributes, null price after enrichment)
3. Derives `minPriceCents`, `maxPriceCents`, `inStock`, and `attributes` from surviving children

A variable product needs at least 2 surviving variations to proceed. One surviving variation collapses to effectively a simple product and is flagged.

### Null prices are intentional

If neither WooCommerce nor Veeqo has a price for a product, `minPriceCents` / `maxPriceCents` are left `null`. This is intentional — used to flag products needing attention before going live, not a data error.

### Deferred fields

`description`, `summary`, and `categories` are left null/empty through most of the pipeline and only populated in later steps (MCP server for content, categorize scripts for categories). This keeps early validation focused on fields available early and avoids validating content that hasn't been generated yet.

### No fallbacks in import

The import script accesses all fields directly — no `.get()`, no defaults. A `KeyError` means something slipped through validation upstream and needs to be fixed there, not papered over in the import. This keeps the import honest about what the validators actually guarantee.

### Parent category products

Products are assigned to leaf categories only (e.g. `brands > bikerarmour > foam-goggles`), not to parent nodes (`bikerarmour` or `brands`). When a server endpoint filters products by a parent category, it needs to recursively include products from all descendant categories — an exact `category_id` match returns nothing for parent nodes.

### No-op deletes

`DELETE FROM brands WHERE slug = X` when that brand doesn't exist yet returns 0 rows and raises no error. This is why the FK violation bug never triggered until bikershades had actually been partially imported — every prior delete was a no-op.

---

## Attribute Normalization

WooCommerce had inconsistent attribute names across brands. Merged to canonical names:

| Original | Canonical |
|---|---|
| Choose Color (REQUIRED), Choose Color, Color, Choose Frame Color (Required), Choose Bandana Color | `color` |
| Choose Power | `power` |
| Qty, Choose Quantity, Cleaner Spray Bottle QTY, Qty (packs of 3) | `quantity` |
| Choose Transition Type | `transition` |
| Choose Size | `size` |

---

## Products Removed From Pipeline

Hardcoded in `remove_items.py` (no longer reads from an external markdown file).

**BikerShades:**
- `LABEL-FEDEX2DAYS`, `LABEL-UPSOVERNIGHT` — shipping service listings, not products
- `vshield-flip-up-face-shield-w-25-mm-pet-transparent-visor` — conflicting quantity attributes
- `7-eye-replacement-foam` — third-party brand, discontinued per owner
- `wiley-x-saint-for-rx-sm-lg` — Rx prescription service listing
- 26 simple products with `-RX` SKUs — Rx prescription service listings

**Sunglass Monster:**
- `prosport-vshield-flip-up-face-shield-w-25-mm-pet-transparent-visor` — conflicting quantity attributes

---

## AI Content Generation

Claude Desktop + a local MCP server (`mcp/server.py`) generated all product descriptions and summaries. The MCP server exposed three tools: `get_next_product`, `save_content`, and `get_progress`. `save_content` enforced strict validation rules (word count, banned phrases, bullet format) and returned `VALIDATION FAILED` with reasons if content didn't pass — Claude Desktop retried until it passed.

**Known issue:** if two products share a slug, `save_content` always saves the first match. The second item loops forever. Fix: manually copy the saved content from the first duplicate to the second in the JSON file.

---

## Final Import Counts (All Three Brands)

| Table | Count |
|---|---|
| brands | 3 |
| categories | 60 |
| products | 870 |
| product_images | 3108 |
| variations | 9633 |
| variation_images | 2085 |
| description_images | 43 |
| product_description_images | 1106 |

All gaps from source data are accounted for by confirmed 404s from the source servers (1 monster product image, 78 bikershades images).
