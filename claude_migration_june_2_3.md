# Migration Learnings (June 2, 2026)
_Everything built and learned across the BikerShades → proSPORT Sunglass Monster migration project._

---

## What was built

A multi-brand WooCommerce → Supabase migration pipeline. Takes raw product and variation data from a WooCommerce store via the admin REST API, cleans and normalizes it, downloads and re-uploads images to Supabase Storage, and imports everything into a Supabase Postgres database.

**32 scripts total:**
- 13 pipeline scripts (fetch → slim → validate → reshape → create → categorize → download → upload → import)
- 5 analysis scripts (field coverage and stats per stage)
- 7 audit scripts (counts and data quality per stage)
- 7 verify scripts (cross-check DB/file state vs source per stage)

**ProSport run-1 results:** 166 products, 1,713 variations, 12 categories, 411 images — all verify scripts PASSED.

---

## WooCommerce Admin REST API

### Store API vs Admin REST API

The old BikerShades pipeline used the unauthenticated Store API (`/wp-json/wc/store/v1/`). This was lossy:
- Variation images came as an `images[]` array (max 1 image)
- Stock was a binary `is_in_stock` boolean — no real quantities
- Prices were nested inside a `prices{}` object

The new pipeline uses the **authenticated Admin REST API** (`/wp-json/wc/v3/`) with HTTP Basic Auth (consumer key + secret). This returns:
- `image` — singular object `{id, src, name, alt}` — the variation's featured image
- `stock_quantity` — real integer, valid only when `manage_stock: true`
- `regular_price`, `sale_price` — top-level strings (not nested)
- Full attribute data: `[{id, name, slug, option}]` where `option` is the term name

### Authentication

HTTP Basic Auth. Consumer key as the username, consumer secret as the password. Credentials stored in `.env` per brand (`PROSPORT_CONSUMER_KEY`, `PROSPORT_CONSUMER_SECRET`). Passed via `requests.get(url, auth=(key, secret))`.

### Pagination

WooCommerce paginates all list endpoints. Standard pattern:
```python
page = 1
while True:
    resp = requests.get(url, params={"per_page": 100, "page": page}, auth=auth)
    batch = resp.json()
    if not batch:
        break
    results.extend(batch)
    page += 1
```
Fetching variations requires one request per product (per page), not one request for all variations.

### on_sale is unreliable at the product level

`on_sale=True` on 170/171 products — WooCommerce propagates this from variations. 154 variable products have `on_sale=True` with no `sale_price` at the parent level. The reliable check is: `sale_price` is set AND `float(sale_price) < float(regular_price)`. This is the rule applied at reshape.

### stock_status is the source of truth for stock

`stock_quantity` is only valid when `manage_stock=True`. For products/variations where `manage_stock=False`, `stock_quantity` is null. Use `stock_status` (`"instock"` / `"outofstock"`) as the source of truth; use `stock_quantity` as supplementary real data when available.

### Variation images are singular, not an array

The admin API returns `image` (singular object) for variations, not `images[]`. At reshape, normalize to `images: [image]` if image exists, `images: []` if not. This matches the DB schema which stores variation images as rows in a `variation_images` table.

---

## Pipeline architecture

### Why each stage exists

| Stage | Purpose |
|-------|---------|
| fetch | Get raw data from WooCommerce, save as-is |
| slim | Extract only relevant fields — reduces noise for all downstream scripts |
| validate | Flag records that will fail import (missing SKU, bad price, draft status, etc.) |
| reshape | Normalize to the DB schema: strip HTML, prices to cents, clean attribute names |
| create_items | Embed variations into products; ownership checks; image dedup; min/max price |
| categorize | Assign normalized `[[{level, name, slug}]]` category paths |
| download | Download all images locally before touching Supabase |
| upload | Upload local images to Supabase Storage |
| import | Insert everything into Postgres |

Splitting fetch from slim means raw API responses are always on disk — no need to re-fetch if a downstream script has a bug.

### Crash-safe fetch_variations

Variations take the longest to fetch (one API call per product, per page). The script writes results incrementally to a progress file:
```python
# On startup: load existing results, build set of already-fetched product IDs
# Skip products already in the file
# After each product: append to file immediately
```
If the script crashes or is killed, re-running picks up from where it left off.

**Caveat:** if a product returns HTTP 503 and is written as 0 variations, the crash-safe logic will skip it on re-run (it's already in the file). Fix: manually remove that product's entry from the JSON, then re-run — the retry will fetch it fresh.

### Variations are fetched only for validated products

`fetch_variations.py` reads from `validate-products/products.json`, not `slim-products/`. This means the 2 flagged products (draft + no-SKU) never have their variations fetched — no wasted API calls, and their variation IDs never pollute the dataset.

### create_items ownership check

When embedding variations into products, the script checks that each variation's `parent_id` matches the product's `id`. Mismatches are flagged. The 3 flagged products (2145, 2211, 1208) failed because their variations were flagged at validate — the variations don't exist in the dataset, so create_items can't embed anything and flags the product.

### Image dedup

At create_items, variation images are compared to the product's gallery. If a variation's image URL is already in the product gallery, the variation's `images` array is cleared to `[]` — no point storing duplicates. 97.9% of ProSport variations end up with `images: []` (1,677/1,713). Only 36 variations carry a unique image.

---

## Shared config module

### The problem

Every script needed `BRAND_CONFIG`, `load_brand()`, and the `BASE` path. Duplicating these across 28 scripts meant any change required editing 28 files.

### The solution: `config/__init__.py`

`new-migration/config/__init__.py` is the single source of truth. Making a folder into a Python package is as simple as adding `__init__.py` — when you `import config`, Python runs that file.

```python
# config/__init__.py
from pathlib import Path
from dotenv import load_dotenv
import os

BASE = Path(__file__).parent.parent  # new-migration/
load_dotenv(dotenv_path=BASE.parent / ".env")

BRAND_CONFIG = {
    "prosport": {
        "slug": "prosport",
        "brand_name": "proSPORT Sunglasses",
        "base_url": "PROSPORT_BASE_URL",
        "consumer_key": "PROSPORT_CONSUMER_KEY",
        "consumer_secret": "PROSPORT_CONSUMER_SECRET",
    },
}

def load_brand():
    key = os.getenv("BRAND")
    config = BRAND_CONFIG.get(key)
    if not config:
        raise SystemExit(f"Unknown BRAND={key!r}. Add it to BRAND_CONFIG.")
    return key, config
```

`load_dotenv` is called here so scripts don't need to call it themselves.

### Why BASE needs two `.parent` hops

Scripts live in `new-migration/pipeline/`, `new-migration/analysis/`, etc. — one level deep. `config/__init__.py` lives in `new-migration/config/` — also one level deep. So `Path(__file__).parent` is the `config/` folder, and `.parent.parent` is `new-migration/`. One hop would give you `config/`, not `new-migration/`.

### sys.path for imports

Scripts live in subdirectories, so Python doesn't automatically include `new-migration/` in `sys.path`. Each script adds it explicitly:
```python
import sys
sys.path.insert(0, str(Path(__file__).parent.parent))
from config import BASE, load_brand
```

### Importing non-`__init__` files from a package

If you have `config/helpers.py`, import it as:
```python
from config.helpers import some_function
```
The `__init__.py` import (`from config import X`) only gives you what's defined or imported in `__init__.py` itself.

### Adding a new brand

One dict entry in `config/__init__.py`, plus the corresponding env vars in `.env`. No other files need to change.

---

## QA framework

### Two types of QA scripts per stage

**Audit scripts** check the output file itself — row counts, field presence, data distribution. They are standalone and don't cross-check against other stages. Good for catching structural issues.

**Verify scripts** cross-check against the previous stage's output (or against the DB after import). They confirm nothing was lost or corrupted across the transformation.

### What verify_import checks

- Product count matches source
- All source SKUs present in DB, no extra SKUs
- Variation count matches
- All variation SKUs present
- All product and variation fields match source exactly
- Every product has at least one category
- Category leaf slugs match source
- Product image count matches
- Variation image count matches
- No duplicate product slugs

### When to run which

- **Analysis** — after fetch and slim stages, to understand the data before writing reshape logic
- **Audit** — after every pipeline stage, to confirm the output looks structurally correct
- **Verify** — after reshape, create, categorize, download, upload, and import, to confirm nothing was lost or corrupted

---

## Data insights — proSPORT Sunglasses

### Product structure

- 171 fetched, 154 variable, 17 simple
- 170 published, 1 draft (a junk copy — flagged and dropped)
- Every product has at least one image; parent gallery averages 3.4 images per product
- 22 products have no description, summary, or description images at all
- 88 products have `total_sales=0` — treat as unknown, not confirmed zero

### Variation structure

- 1,758 fetched across 153 variable products (16 simple have no variations)
- Max 88 variations on a single product
- 1,740 passed validation; 18 flagged (17 with spaces in SKUs across 2 products, 1 with no price)

### Categories

WooCommerce always appends "Uncategorized" alongside real categories — every product has it. Strip at reshape. 21 products had only "Uncategorized" with no real category — assigned manually via hardcoded dict in `prosport_categorize_items.py`.

Final category tree: 4 root categories, 8 leaf categories. Each product assigned one leaf path.

### Attributes

9 attribute names, but only 2 are real variation axes:
- **Color** — 147 products, 208 unique color options
- **Power** — 65 products (readers/bifocals), values 1.00–4.00 in 0.25 increments

The other 7 (`Activity`, `Frame Color`, `Frame Styles`, `Gender`, `Lens Color`, `Lens Type`, `Material`) are filter-only attributes — they appear on a handful of products with 1-2 options and don't drive variation combinations.

### Pricing

All 166 imported products are on sale (sale discount 15–47%, median 41%). 141 products have flat pricing across all variations; 9 (the Power readers) have tiered pricing by strength.

### Stock

147 products report `stock=1` as a WooCommerce fallback (it's the aggregate when `manage_stock=False`). Only 8 products have real stock quantities at the product level. Variation-level stock is more reliable: 1,332 variations have real quantities (total 9,497 units), 299 are out-of-stock.

---

## Lessons learned

### Background piping kills long-running Python scripts

Running `python script.py 2>&1 | tail -10 &` will kill the Python process once tail's pipe buffer fills and tail exits. The script gets a broken pipe signal and terminates. Only 77/153 products were fetched this way before the process died. Fix: run long scripts in the foreground without piping, or redirect to a log file (`> out.log 2>&1 &`) which doesn't close.

### 503 errors need manual intervention with crash-safe scripts

When a product returns HTTP 503 during `fetch_variations`, the crash-safe logic writes it as an entry with 0 variations. On re-run, it sees the entry already exists and skips it — the error is silently "fixed" into missing data. Must manually remove those entries from the JSON before re-running.

### Dead fields are common in WooCommerce responses

~40 fields in the raw product response carry zero information: `virtual`, `downloadable`, `tax_status`, `backorders_allowed`, `average_rating`, `rating_count`, `tags`, `brands`, `default_attributes`, `dimensions` (object exists but all subfields are empty strings). Identifying these at the analysis stage before writing the slim script prevents them from polluting downstream data.

### Stale field references break analysis scripts

`analyze_slim_products.py` referenced `related_ids` — a field that existed in the raw API response but was dropped at the slim stage. This caused a `KeyError` on first run. Analysis scripts must only reference fields that are actually present at their stage. Read the slim script before writing the analysis script.

### Product `on_sale` is not trustworthy for variable products

WooCommerce sets `on_sale=True` on a product whenever any variation is on sale. 154 variable products have `on_sale=True` with no `sale_price` at the parent level. The only reliable `on_sale` check is at the variation level: `sale_price` is set and numerically less than `regular_price`.

### Descriptions embed hardcoded image URLs from the old store

Product descriptions contain `<img>` tags pointing to `sunglassmonster.com`. These are "description images" — inline images in the product description HTML. The pipeline strips the description HTML and extracts these as separate `descriptionImages: [{src, name}]` objects, which get downloaded and re-uploaded to Supabase Storage. This keeps descriptions portable across stores.

### WooCommerce parent SKUs are meaningless on variable products

The parent product has a `sku` field, but for variable products it's meaningless — all real commerce happens through variations which have their own SKUs. Nullify parent SKUs on variable products at reshape so nothing accidentally uses them.

### Spaces in variation SKUs cause silent failures downstream

17 variations across 2 products had spaces in their SKUs (e.g., `"PS-123 A"` instead of `"PS-123A"`). WooCommerce accepts them, but they fail DB uniqueness constraints or cause lookup mismatches. Validate for this explicitly — `sku.strip() != sku` or `" " in sku` — and flag at validate_variations.

---

## Supabase integration

### Storage

Images are uploaded to a named bucket (e.g., `prosport`) via the Supabase Storage API. The upload script maintains an `upload_map.json` that maps local file paths to Supabase public URLs — crash-safe, same pattern as download.

The import script resolves image URLs from `upload_map.json` at import time — the pipeline never hardcodes storage URLs into item data.

### Import order matters

Tables have foreign key constraints. Import order: brands → categories → products → variations → product_images → variation_images → description_images → product_categories. Inserting out of order causes FK violations.

### Verify after import, not before

The verify script reads from the live DB and compares against the source item JSON. Running it before import is pointless. Running it after import is the definitive check that everything landed correctly.

---

## Python patterns used

### `Path(__file__).parent` for portable paths

Never hardcode absolute paths. `Path(__file__).parent` gives the directory containing the current script, regardless of where it's run from. All file paths derived from this.

### `__init__.py` makes a folder importable

Create `mypackage/__init__.py` and Python treats `mypackage/` as a package. `import mypackage` runs `__init__.py`. Anything defined or imported there is available as `mypackage.X` or `from mypackage import X`.

### Incremental JSON writes for crash safety

Instead of writing the full output at the end, append each result to a list and write the full list to disk after each item. On restart, load the existing list and skip items already processed. This makes any multi-hour fetch resumable after a crash or kill.

### Bulk script transformation with regex

When the same pattern needs to change across 28 files, write a one-off Python script using `re.sub` to make the substitution programmatically. Regex replacements on multi-line blocks use `re.DOTALL`. Much faster and less error-prone than manual edits across dozens of files.
