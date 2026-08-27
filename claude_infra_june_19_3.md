# Migration Learnings (June 19, 2026)

Multi-brand WooCommerce → Supabase product catalogue migration. Three brands: prosport, bikershades, monster.

---

## Pipeline Overview

Each step reads from the previous step's output directory. Run from repo root with `BRAND=<brand> python final-migration/pipeline/<script>.py`.

```
fetch_items.py
  → reshape_items_1.py       (shape + WC values)
  → reshape_items_2.py       (Veeqo enrichment, variable product derivation)
  → validate_items.py        (non-deferred field checks)
  → categorize_items.py      (category tree)
  → normalize_image_names.py (image name normalization)
  → decode_entities.py       (HTML entity decode, option slugify, color hex)
  → remove_stock.py          (strip stock fields)
  → validate_items_final.py  (full pre-import validation)
  → download_items.py
  → upload_items.py
  → import_items.py
```

Output for each step lands in `final-migration/data/{brand}/{step-name}/`.

---

## DB Schema Design

### Brand reference: `brand_slug` not `brand_id`

All tables reference `brands(slug)` as a `text` FK instead of `brands(id)` (UUID). This avoids the lookup step in import and makes queries more readable. The slug is stable and unique per brand.

### Stock removed from schema

`stock` and `in_stock` were removed from `products`; `stock` removed from `variations`. The DB is the source of truth for what's sold — stock is managed externally (Veeqo) and not stored here.

### `description_images` as a brand-scoped image bank

A single description image URL used by multiple products is stored once in `description_images` (unique on `brand_slug, src`) and linked via the `product_description_images` join table. Avoids duplication across products that share size charts, etc.

### No `TRUNCATE` on re-import

Re-import does explicit ordered deletes (`variation_images → product_description_images → product_categories → product_images → variations → products → description_images → categories → brands`) rather than `TRUNCATE`. This leaves other brands untouched.

### `ON DELETE CASCADE` exists but isn't relied on

Schema has cascade FKs for correctness on fresh deploys. Import does explicit ordered deletes regardless — don't rely on cascade for re-import.

---

## Key Pipeline Decisions

### Variable products

- SKU is null at product level; SKU checks apply at variation level only.
- `minPriceCents`, `maxPriceCents`, `inStock` (removed), and `attributes` are all derived from surviving child variations in `reshape_items_2.py`.
- Bad variations (not in Veeqo, not an object) are dropped silently. A variable product needs at least 2 surviving variations to proceed.

### Deferred fields

`description`, `summary`, and `categories` are left null/empty in reshape and filled in a later step. `validate_items.py` skips these; `validate_items_final.py` checks them.

### Null price is intentional

`minPriceCents` / `maxPriceCents` being null means neither WC nor Veeqo had a price. Used to flag products needing attention — not a bug.

### Import has no fallbacks

All product/variation fields are accessed directly. A `KeyError` means a validation gap upstream, not a case to handle in import.

---

## `decode_entities.py`

### What it does

1. HTML entity decode: `html.unescape()` on product names, attribute names, and all option/variation values.
2. Slugify all attribute options (product level) and variation attribute values (lowercase, non-alphanumeric → hyphens).
3. For `color` attributes only: derive a hex value from the slug.

### Option structure

Product-level attribute options:
```json
{ "option": "Black Silver", "slug": "black-silver", "value": "#c0c0c0" }  // color attr
{ "option": "3.00", "slug": "3-00" }                                        // non-color attr
```

Variation-level attribute objects:
```json
{ "name": "color", "option": "Black Silver", "slug": "black-silver", "value": "#c0c0c0" }
{ "name": "power", "option": "3.00", "slug": "3-00" }
```

Key: `value` is only present when `name == "color"`. `option` is the display name, `slug` is the URL-safe version.

### Color derivation

Scan slug words right-to-left, return hex for the first word found in `COLOR_MAP`. Fallback: `#ffffff`.

Example: `"black-silver"` → scan `["silver", "black"]` → first match is `"silver"` → `#c0c0c0`.

Lens color appears last in WooCommerce naming ("Frame Color Lens Color"), so right-to-left gives lens color priority — which is what's displayed.

### COLOR_MAP design

Only actual colors (hex values). Removed shade/finish/tech descriptors that aren't colors: `smoke`, `hd`, `lava`, `pearl`, `titanium`. If a word isn't in the map it's skipped — no need for a separate exclusion set.

---

## `remove_stock.py`

Separate step (own output dir `remove-stock/`) that strips `stock` and `inStock` from products and `stock` from variations. Kept separate from `decode_entities.py` so each step has a single responsibility and the pipeline remains composable.

---

## `validate_items_final.py`

Full pre-import validator. Reads from `remove-stock/items.json`.

Checks include:
- All non-deferred fields present and correctly typed
- Slug format
- Duplicate attribute combinations within a product
- Image uniqueness
- Price consistency
- Option field types: `option`, `slug` must be str; `value` must be str when present; attribute `name` must be str — checked at both product and variation level
- Variation options must reference declared product attributes (object comparison minus `name` key)

Products accumulate all failures before being flagged. One failure is enough to flag.

**Known flagged products (pre-existing, not errors):** duplicate attribute combinations in 3 prosport bifocal products — these exist in WooCommerce and carry through.

---

## `import_items.py`

- Uses `OFFICE_DATABASE_URL` (Supabase session mode pooler) — direct connection hostname fails to resolve on some networks (IPv6/DNS issue).
- Brand inserted by `(name, slug)` — no UUID needed since all downstream tables reference `brand_slug` (text).
- Categories built lazily via `ensure_category()` as products are processed; keyed by `(parent_id, slug)`.
- Commits every 50 products.

---

## Image Pipeline

- **Download filenames**: everything after `/uploads/` or `/gallery/` in the URL path with `/` replaced by `_`. Guarantees uniqueness within a brand without date-prefix ambiguity.
- **Image name field**: normalized from URL basename stem with `-` and `_` replaced by spaces (done in `normalize_image_names.py`).
- **`url_map.json`** is the source of truth mapping original URL → local filename. SKU subfolders are unnecessary.
- Unresolvable images (404s from source site) are skipped with a WARN and counted. They're pre-existing — not pipeline errors.

### Final image counts (skipped)

| Brand | Skipped |
|---|---|
| prosport | 0 |
| bikershades | 78 |
| monster | 1 |

---

## Final Import Counts

| Brand | Products |
|---|---|
| prosport | 150 |
| bikershades | 464 |
| monster | 256 |
| **Total** | **870** |

---

## Attribute Name Combinations (across all brands)

| Attributes | Count |
|---|---|
| `[color]` | 519 |
| `[color, power]` | 223 |
| `[color, transition]` | 73 |
| `[]` (no attributes) | 50 |
| `[quantity]` | 2 |
| `[power]` | 1 |
| `[size]` | 1 |
| `[color, quantity]` | 1 |

---

## Brand Config: Two Slug Fields

- `slug` — pipeline key, data directories, storage bucket path (`prosport`, `monster`, `bikershades`)
- `brand_slug` — DB slug, Supabase bucket name (`prosport-sunglasses`, `sunglass-monster`, `bikershades`)

Never conflate these. `slug` is for local paths; `brand_slug` is what goes into the DB.
