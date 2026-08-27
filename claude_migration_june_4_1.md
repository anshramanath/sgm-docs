# Migration Learnings (June 4, 2026)
_Updated: 2026-06-05_

---

## What we built

A multi-brand WooCommerce → Supabase migration pipeline. Three brands (proSPORT, Sunglass Monster, BikerShades), each running through the same staged pipeline:

```
fetch → slim → validate → reshape → create-items → categorize → download → upload → import
```

Each stage reads the previous stage's output. All output is scoped to `data/{slug}/` so brands don't collide. The active brand is selected via `BRAND` in `.env`.

---

## WooCommerce API

### Admin API vs Store API
- **Admin API** (`/wp-json/wc/v3/`) requires consumer key/secret (HTTP Basic Auth) and returns complete data: real `stock_quantity` ints, full price fields at top level, complete variation images
- **Store API** (old BikerShades pipeline) was unauthenticated but lossy — binary `is_in_stock`, prices inside a `prices{}` object, variation images limited to 1

### Variation image field
- Admin API returns `image` as a singular object `{id, src, name, alt}`, not an array
- `image` can be `null` — some variations (photochromic lens variants on BikerShades) have no image set in WooCommerce
- Only flag if an image IS present but malformed (no src, bad URL, empty name)

### Prices
- Top-level strings: `regular_price`, `sale_price`, `price`
- Not inside a `prices{}` object (that's the Store API shape)

### Variable product SKU
- WooCommerce appends `-0` to the parent product SKU to prevent duplicates when two products share the same variation base
- Sometimes appends more after `-0`: `-0-XT` (photochromic), `-0-SM` (standard lens), `-0-COMBOS`, `-Master-MFB` etc.
- The pattern is: everything from `-0` onwards is WooCommerce's dedupe suffix, not part of the model identifier
- **Do not derive parent SKU from variation common segments** — stripping WC's suffix collapses distinct products into the same SKU (verified: 200+ duplicate flags surfaced)
- Keep WC product SKU as-is

### Variable product stock/weight/dims
- WooCommerce aggregates stock across variations — unreliable, often wrong
- Set `stock = null` for all variable products in reshape; variation-level stock is the source of truth
- Same for `weight` and `dimensions` — null out for variable products

---

## SKU patterns

### Variation SKU structure
`{modelBase}-{frameColor}-{lensType}-{strength?}-{suffix?}`

Examples:
- `FM85X11MN-BK-CL-MFB` — Magnum, black frame, clear lens
- `RE14X881GN-GD-GSM-100-MFS` — reader, gold frame, gray smoke mirror, +1.00 strength
- `CPRE5X07ZA-BL-CL-150` — computer pre-reader, blue frame, clear lens, +1.50

### Parent product SKU = variation base + `-0` (usually)
- Strip `-0` and everything after to get the model base
- That base is always a prefix of every variation SKU in the product
- Guaranteed by the base SKU mismatch check at validate time

### Base SKU mismatch check
- All variations within a product must share the same first dash-segment (`sku.split('-')[0]`)
- This catches: combo products mixed with regular, polarized `CAPL` variants mixed with non-polarized `CA` in one WC product
- BikerShades: 49 products flagged (1,114 variations) — heavy `CAPL`/`CA` mixing
- ProSport: 11 products, Monster: 17 products

### Combo products
- `CPR` prefix = two pairs for one price (Monster)
- Other combos: `CFM*-COMBOS`, `CGO*-COMBO` (BikerShades) — explicitly named in product title
- Caught by base SKU mismatch check when mixed with regular variations, or identifiable by name/suffix
- Dual color format: `HDXCL` = HD + Clear (two lenses in one SKU segment)

### WooCommerce SKU junk patterns
Arbitrary suffixes on parent product SKUs that can't be cleanly stripped:
- `-Master-MFB` (X-Loop master products)
- `-COMBOS` without `-0`
- Casing changes: `WileyX-` vs `WX-`
- Typos: `CA530X93GMM` when variations are `CA530X93GM`
- Size in parent SKU: `BF4X05LP-SM-0` but variations are `BF4X05LP-BK-SM-...`

---

## Data quality patterns by brand

### ProSport
- Cleanest data of the three
- 0 null variation images
- 0 duplicate SKUs
- 4 products with mismatched parent SKU (pre-existing WC errors)
- 11 base mismatch products (9 distinct products lost)

### Sunglass Monster
- Also clean — 2 flagged variations (1 SKU space, 1 missing price)
- Same 4 parent SKU mismatches as ProSport (cross-brand shared products)
- 17 base mismatch products

### BikerShades
- Heaviest junk: Rx add-ons, USPS return labels, face masks, warranty products, BikerArmour (body armor)
- 93 products flagged at validate-products
- Photochromic variations with `image: null`
- 6 variations sharing the same `LABEL-USPSRETURN-1-1` SKU
- Sells same frame in multiple lens types as separate WC products (using `-0-XT`, `-0-SM` etc.)
- 49 base mismatch products — polarized (`CAPL`) vs non-polarized (`CA`) variants grouped in one WC product
- 24 parent SKU mismatches after stripping `-0` and onwards

---

## Pipeline robustness improvements

### Validate time
- **Attribute name collision detection** — simulate `clean_attr_name` at validate time, flag if two attributes would collide after cleaning
- **Base SKU mismatch check** — all variations in a product must share the same first dash-segment; flags combos and mixed polarity products
- **Null image allowed** — variation `image: null` is valid; only flag if image is present but malformed
- **WooCommerce slug validation** — just check non-empty string; WC slugs diverge from names (apostrophes, truncation)
- **Empty attribute options OK** — `create_items.py` fills them from variation data

### Reshape time
- **Variable product stock = null** — WC aggregate is misleading
- **Variable product weight/dims = null** — variation-level is source of truth
- **All-or-nothing dimensions** — if only some of length/width/height present, null all three; print `[INFO]`
- **`sortOrder` on all image objects** — 1-indexed, set at reshape, read at import (no generation at import time)
- **`weightUnit` / `dimensionUnit` = null** — present as explicit fields, not inline defaults

### Create-items time
- **Min/max price derived from variation `regularPriceCents`** — not from WC product price
- **Sale flag computed from variations** — `any(v["sale"] for v in filled)`
- **Image dedup** — variation images that duplicate a parent image are removed
- **Attribute expansion + prune** — variation options fill parent attribute options; unused options pruned

### Import time
- **Pure insertion** — no field generation or fallbacks in `import_items.py`
- **All values from categorize-items output** — sort orders, units, prices all come from data

---

## QA approach

Every stage has two scripts:
- `audit_*.py` — stats view (counts, ranges, distributions). Run to understand the data.
- `verify_*.py` — correctness checks (cross-check reshaped data against source). PASS/FAIL.

Verify scripts check:
- Field presence and absence (expected fields in, dropped fields out)
- Pass-through fields unchanged
- Derived fields correct (prices to cents, HTML stripped, sortOrder sequential)
- Structural invariants (variable price = null, stock = null)
- Global uniqueness (SKU dedup across all products + variations)

---

## Category tree

- Brand-specific `{brand}_categorize_items.py` — category mapping is subjective
- Every node in every path gets `sortOrder` stamped (1-indexed, resets per parent)
- Import reads `node["sortOrder"]` directly — no enumerate at import time
- ProSport: 12 categories, 197 leaf assignments

---

## Crash safety

- `download_items.py` — crash-safe via `download_map.json`; re-run picks up where it left off
- `upload_items.py` — crash-safe via `upload_map.json`

---

## Multi-brand pattern

One pipeline, one config module, one `BRAND` env var:
```python
BRAND_CONFIG = {
    "prosport": {...},
    "monster": {...},
    "bikershades": {...},
}
```
Adding a new brand = one dict entry + credentials in `.env`.

---

## Counts

| Brand | Products | Variations | Items | Flagged |
|-------|----------|------------|-------|---------|
| proSPORT (run 2) | 169 validated | 1,740 | 166 | 3 |
| Sunglass Monster | 267 validated | 2,673 | 265 | 2 |
| BikerShades | 611 validated | 4,839 | 553 | 58 |

---

## Things to do differently next time

- **Validate SKU format early** — the base mismatch check at validate time was the right call; catch data issues before they propagate
- **Don't trust WC parent product SKU** — it's a WC-generated dedupe string, not the canonical model ID
- **Admin API from the start** — the Store API (old BikerShades pipeline) lost too much data
- **Brand-specific junk patterns** — each brand has its own class of junk products (Rx add-ons, body armor, face masks); review raw fetch output before assuming clean data
- **Image null is common** — don't assume all variations have images; photochromic/transition products often don't
