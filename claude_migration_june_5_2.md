# Migration Project — Learnings & Build Notes (June 5, 2026)
_Covers: pipeline architecture, per-brand runs, robustness improvements, Veeqo analysis_
_Last updated: 2026-06-05_

---

## 1. Project Overview

Multi-brand WooCommerce → Supabase migration pipeline. Three brands:
- **prosport** — proSPORT Sunglasses
- **monster** — Sunglass Monster
- **bikershades** — BikerShades

Uses the **authenticated WooCommerce admin REST API** (`/wp-json/wc/v3/`) with consumer key/secret. The active brand is selected via `BRAND` env var. All scripts import brand config from `new-migration/config/__init__.py`.

---

## 2. Pipeline Architecture

### Products
1. `fetch_products.py` — fetches all products via admin API → raw JSON
2. `slim_products.py` — extracts only the fields we care about
3. `validate_products.py` — validates each field, flags bad records

### Variations
4. `fetch_variations.py` — fetches all variations for each validated product
5. `slim_variations.py` — extracts relevant fields
6. `validate_variations.py` — validates each variation, flags bad records

### Reshape → Import
7. `reshape_products.py` — normalizes: strips HTML, prices → cents, cleans attribute names, descriptionImages → `{src, name, sortOrder}` objects
8. `reshape_variations.py` — same conventions, `image` → `images[]` array with sortOrder
9. `create_items.py` — embeds variations into products, deduplicates images, computes min/max price, expands attribute options from variation data
10. `veeqo_items.py` — gates on Veeqo SKU presence; flags items whose SKUs are absent
11. `{brand}_categorize_items.py` — brand-specific; assigns category paths as `[{level, name, slug, sortOrder}]` nodes
12. `download_items.py` — downloads all images locally (crash-safe via `download_map.json`)
13. `upload_items.py` — uploads to Supabase Storage (crash-safe via `upload_map.json`)
14. `import_items.py` — inserts into Supabase DB; pure insertion, no inline defaults

All output scoped to `data/{brand_slug}/{stage}/`.

---

## 3. Key Technical Decisions

### Admin API vs Store API
Old pipeline used the unauthenticated Store API. New pipeline uses the authenticated Admin REST API. Key differences:
- Stock: binary `is_in_stock` bool → real `stock_quantity` int
- Prices: inside `prices{}` object → top-level strings
- Variation images: `images[]` array → singular `image` object
- Auth: none → HTTP Basic (consumer key + secret)

### `clean_attr_name()` rules
Strips four things from raw WooCommerce attribute names before using them as DB column identifiers:
1. `for Filter` suffix (case-insensitive)
2. `Choose ` prefix (case-insensitive)
3. `Filter by ` prefix (case-insensitive)
4. Parentheticals like `(REQUIRED)` — using `\s*\(.*?\)` regex

Defined in `reshape_products.py` and **replicated verbatim** in both `validate_products.py` and `validate_variations.py` so collision detection stays in sync with reshape behavior.

### Attribute name collision detection
Within a single product or variation: if two attribute objects' names both clean to the same string → flag it. Catches reshape-time collisions at validation time, before any data transformation happens.

Previously this was coupled to an `ATTR_OVERRIDES` dict (raw name → canonical name) that resolved collisions before the generic rules ran. Both were removed together. Collision check was re-added standalone with no override dependency.

### Variable product weight/dimensions/stock = null
WooCommerce provides aggregate-level stock and physical dimensions for variable products, but these are unreliable — variation-level is the source of truth. `reshape_products.py` nullifies all three for variable products.

### All-or-nothing dimensions
If only some of length/width/height are present after reshape, all three are nullified and an `[INFO]` is printed. Applies to both simple products and variations. Prevents partial dimension sets from reaching the DB.

### sortOrder on everything
Every image in `images[]`, `descriptionImages[]`, and variation `images[]` gets a 1-indexed `sortOrder` during reshape. Every category node gets a `sortOrder` from the brand-specific `SORT_ORDER` dict. Import reads all of these directly — no generation at import time.

### Import as pure insertion
`import_items.py` has no inline defaults, no fallbacks, no field generation. All values come from categorize-items output. If a field is missing there, it's missing in the DB — intentional, so bugs surface early rather than silently defaulting.

### WooCommerce slug validation
Removed the `slugify(name) == slug` check. WooCommerce slugs diverge from names in real data (apostrophes, truncation, etc.). Just check non-empty string.

### Empty attribute options OK
`validate_products.py` no longer flags empty `options[]` arrays on variable products. `create_items.py` fills them from variation data — variable products legitimately have no options at the product level.

### Base SKU consistency check — removed
Was comparing the first hyphen-segment across sibling variations. Superseded by the **NNXnn model consistency check**, which extracts the `\d+[Xx]\d+` model number from each variation SKU and flags if siblings have different model numbers. Catches the same real-world cases (mixed frame models, SKU revisions coexisting) more precisely.

### Variable product SKU kept as-is
WooCommerce appends `-0` (or similar) to variable product SKUs to prevent duplicates when two products share the same variation base SKU. Deriving the product SKU from common variation segments strips this suffix and causes false duplicate flags in `create_items.py`.

### Non-product SKU filtering (BikerShades)
`JUNK_SKU_PREFIXES`, `JUNK_SKU_SUFFIXES`, and `JUNK_SKU_CONTAINS` in `create_items.py` filter WooCommerce entries that aren't real products:
- `LABEL-` prefix → shipping labels
- `TRYB4UB-` prefix → Try Before You Buy deposits
- `-RX` anywhere in SKU → Rx frame listings (reimplemented as a feature, not a product)

### Null variation image allowed
`validate_variations.py` permits `image: null`. BikerShades photochromic variations have no featured image set in WooCommerce. Only flags if an image object is present but malformed.

### Veeqo as pipeline gating step
`pipeline/veeqo_items.py` reads `create-items/items.json`, flags items whose SKUs are absent from Veeqo, and writes to `veeqo-items/`. `{brand}_categorize_items.py` reads from `veeqo-items/`. A separate `qa/veeqo_items.py` exists as stdout-only for informational use.

---

## 4. Per-Brand Run State

### ProSport
- 171 fetched → 169 validated → 161 create-items → **154 items, 7 flagged (veeqo)**
- 1,758 variations fetched → 1,669 validated → 1,646 embedded
- Flagged: bifocals/readers entirely absent from Veeqo (Verbena, Bypass, Iris, Gullwing, Brooklyn, Cooper) + Sharkbait Bifocal (7/16 SKUs missing)
- `prosport_categorize_items.py` already written; not yet run

### Sunglass Monster
- 277 fetched → 267 validated → 260 create-items → **255 items, 5 flagged (veeqo)**
- 2,675 variations fetched → 2,560 validated → 2,537 embedded
- Flagged: Verbena (1/63), Cooper Bifocal (4/20), Sharkbait Bifocal (7/16), Mysto (4/4), Nautica (3/3)
- `monster_categorize_items.py` not yet written

### BikerShades
- 704 fetched → 611 validated → 461 create-items → **458 items, 3 flagged (veeqo)**
- 5,993 variations fetched → 5,299 validated → 5,239 embedded
- Flagged: Cabot Photochromic Reader (8/10), Edison Photochromic Reader (8/10), Hawkeye HD Bifocal (20/26)
- `bikershades_categorize_items.py` not yet written
- **Open: Rx frame products** — 10 BikerShades SKUs with `-RX` suffix pass the junk filter because `-RX` appears after the first hyphen segment, not at the start. Options: filter any SKU segment containing `-RX`, or restore `JUNK_SKU_CONTAINS` `-RX` check. Deferred.
- **Open: full pipeline re-run needed** — adding attribute collision check to validate scripts cascades through all downstream stages. Deferred.

---

## 5. Veeqo Analysis

### What it is
`veeqo-analysis/analyze.py` — standalone script (no pipeline dependencies). Reads the latest `.xlsx` or `sellables_export*.csv` from the same directory. Analyzes SKU quality and flags actionable problems.

### Logic decisions

**Spaces in SKUs — not flagged.** Two sub-types exist in the data:
- Informal descriptive names (`"cobra smoke"`, `"clutch full reader 1.0blk"`) — products entered with human-readable names instead of proper codes
- Typo'd real SKUs (`"CRE59X22DE- BKXBK-CL-100-MFS"` — space after hyphen)
Both types still connect and work in Veeqo. Acknowledged; not flagged.

**Duplicates — only flagged if price differs.** A SKU can be listed on multiple channel listings (e.g. two separate eBay listings). Veeqo creates a separate row per listing. Same-price duplicates are expected and not a problem. Only duplicates where prices differ across rows are flagged — that means the same product is selling at different prices on different channels.

**Placeholders — flagged.** Two patterns:
- `none` / `none1`–`none30` — Amazon listings created without a real SKU. Reused across different products, causing collisions and price conflicts.
- All-digit codes (`204150`, `204200`, `204250`, `204300`) — likely vendor/supplier reference numbers.

**Zero price — flagged.** Any SKU with at least one $0.00 row.

**Multiple problems per SKU** — a single SKU can have multiple issues listed. All are shown together.

### Key bug fixed: stock infinity
`qty_on_hand` can contain infinity as a float. Fixed: `int(f) if f != float("inf") else 0`.

### Current findings (Jun 2026 export)
- 25,008 total rows, 111 no SKU, 24,396 unique SKUs
- Clean: 22,332 | Spaces (not flagged): 2,029 | Placeholders: 35
- Duplicates: 431 total | 162 price-conflict (flagged) | 269 same-price (not flagged)
- Zero price: 56 SKUs
- Stock: 40% of rows at zero (unreliable as migration signal), 111,777 total qty

**Price conflict tiers (162 flagged):**
- Large (>$5 gap): FM85X11MN family + CBF79X71TP + FM25X88MN-RD — 27 SKUs
- Moderate ($2–5 gap): BF2X06MT, BF8X64MG, BF8X68VN, BF83X78PH-BK-SM, BF4X33FL, chopper YELLOW, LO91X206CL, Metric S45, 48/96 Rectangle — 55 SKUs
- Small (<$2 gap): all others — 80 SKUs

**Rows without SKU (111):**
Heavy on Rx add-ons (~30 rows), plus operational items (custom phone orders, shipping fees, distributor invoices), Nike/sporting goods, face masks, miscellaneous non-sunglasses. These should be assigned real SKUs or deleted from Veeqo.

---

## 6. Open Items

| Item | Status |
|---|---|
| BikerShades `-RX` SKUs slipping through junk filter | Deferred |
| Full pipeline re-run (attr collision check added) | Deferred |
| `bikershades_categorize_items.py` | Not started |
| `monster_categorize_items.py` | Not started |
| ProSport categorize → download → upload → import | Not started |
| Veeqo Rx add-on rows (111 no-SKU rows) | Owner decision needed |
| Placeholder SKUs (`none*`, all-digit) | Owner decision needed |
| 56 zero-price SKUs | Owner review needed |
| Price-conflicting SKUs (162) | Owner review needed |

