# Bikershades Migration — Full Notes (May 29, 2026)

Complete record of everything built and learned migrating bikershades.com from WordPress/WooCommerce into a new multi-brand Supabase app.

---

## What Was Built

### Scripts (all in `migration/`)

| Script | What it does |
|---|---|
| `fetch_products.py` | Paginates the WooCommerce Store API, saves all 647 products to `products.json` |
| `fetch_variations.py` | Fetches full variation data for every variation ID. Crash-safe — checkpoints per variation |
| `reshape_products.py` | Normalizes raw WooCommerce data into clean app schema, buckets products by issue type |
| `download_images.py` | Downloads all product/variation/description images locally. Crash-safe via `url_map.json` |
| `upload_images.py` | Uploads local images to Supabase Storage. Crash-safe via `upload_map.json` |
| `import_products.py` | Inserts all clean products into Supabase via psycopg2. Wipe-and-restart strategy |

### Database (`migration/db/001_initial_schema.sql`)

9-table Postgres schema:
- `brands`, `categories`, `products`, `product_categories`, `variations`
- `product_images`, `variation_images`, `description_images`
- `admins` (gates access via Supabase Auth)

### Image Infrastructure

- **Bucket:** `product-images` (Supabase Storage, public)
- **Structure:** `clean-products/products/{sku}/`, `variations/{sku}/`, `descriptions/{sku}/`
- **Two mapping files:**
  - `url_map.json` — bikershades URL → local path (written during download)
  - `upload_map.json` — local path → Supabase Storage URL (written during upload)
  - Chain: `bikershades URL → url_map → local path → upload_map → storage URL`

---

## Migration Results

| | Count |
|---|---|
| Raw products fetched | 647 |
| Variations fetched | 6,064 |
| Images downloaded | 2,785 |
| Unique images uploaded | 2,559 |
| Clean products imported | 589 |
| Categories | 63 |
| Brands | 1 (bikershades) |
| Flagged for manual review | 58 |

### Flagged bucket breakdown
| Issue | Count |
|---|---|
| Invalid SKU (null or has space) | 22 |
| No weight | 12 |
| Zero price (internal products) | 8 |
| Missing variation attributes | 7 |
| Duplicate name | 6 |
| No images | 3 |

---

## WooCommerce API — What We Learned

### Products endpoint
`GET /wp-json/wc/store/products?per_page=100&page=N`

- Returns parent products only (not variations) — intentional WooCommerce design
- Paginate until empty page returned
- Prices are **strings in cents** — `"1999"` = $19.99
- `stock_status` is null on every product — actual stock lives in `is_in_stock` (bool)
- `on_sale: true` doesn't mean the product is actively discounted — must also check `sale_price < regular_price`
- `variation` field on stub objects is often an empty string — use `attributes[]` from the stub as source of truth for attribute values instead
- 9 simple products have `has_variations: true` — WooCommerce misconfiguration, handled naturally

### Variations endpoint
`GET /wp-json/wc/store/products/{id}/variations/{variation_id}`

- Must be fetched individually per variation ID — no bulk endpoint
- Contains per-variation images, prices, stock, SKU, weight
- `variation` field is a human-readable string like `"Choose Color: Blue Green Smoke"` — not reliable for parsing, use stub attributes instead

### Data findings
- 75% of products are variable (489/647)
- ~81% of products are on sale
- Description images: 990 total URLs across 587 products + 4 variations, but only ~40 unique images (same diagrams reused everywhere)
- 86 variation images + 10 description images were 404 (deleted from server ~2018)
- Zero product-level images were 404

---

## Schema Decisions

### `attributes` as `jsonb` not `text[]`
**Why:** A flat slug array like `["black-clear", "blue-smoke"]` can't tell you which attribute type a term belongs to. Listing pages need to render color swatches without joining all variations. Grouping by type: `[{name: "color", terms: ["black-clear", "blue-smoke"]}]` makes this a single column query.

**Query example:**
```sql
SELECT * FROM products
WHERE attributes @> '[{"name": "color", "terms": ["blue-smoke"]}]';
```

### `categories` as `[{name, link}]` objects
**Why:** WooCommerce categories have both a slug (`medium-large`) and a URL. Keeping the link enables 301 redirects from old Google-indexed category pages. Using slug as the name keeps it clean for filtering.

### `description_images` as a separate table with nullable FK
**Why:** Description images are embedded in product/variation HTML. They belong to either a product OR a variation, never both. A CHECK constraint enforces exactly one is set:
```sql
check (
  (product_id is not null and variation_id is null) or
  (product_id is null and variation_id is not null)
)
```

### `name` dropped from variations
**Why:** Variation names in WooCommerce are always identical to the parent product name. Storing it is pure redundancy. The variation is identified by its `variation text[]` field (the option combo) and its `sku`.

### `brand_id` on products (kept despite seeming redundant)
**Why:** Redundant with the join through categories, but useful for direct filtering — `WHERE brand_id = X` without joining anything. Accepted redundancy for query convenience.

### `sale` on variations (not just products)
**Why:** Allows marking individual color variants on sale independently. A popular colorway can stay full price while a slow-moving one is discounted.

### `brands` is just `id + name`
**Why:** Slug is derivable in code. No other fields needed — the brand is just an ownership marker on products and categories.

### No signup flow
**Why:** Supabase Auth's signup endpoint is publicly accessible. Without blocking it, anyone can create an account. The `admins` table acts as an explicit allowlist — even if someone creates an account, they can't do anything without a row in `admins`.

---

## Image Deduplication

Description images are the same ~40 diagrams referenced across hundreds of products. During download, `url_map.json` handles dedup automatically — the first download of a URL writes the local path, subsequent hits to the same URL see it's already in the map and skip. Result: one local file per unique URL regardless of how many products reference it.

During upload, `upload_map.json` is keyed by local path — so even if multiple products point to the same local file, it's uploaded once and all references get the same storage URL.

---

## Crash Safety Pattern

All long-running scripts use the same checkpoint pattern:
1. Load existing checkpoint file on startup
2. Skip anything already in the checkpoint
3. Write to checkpoint immediately after each successful operation
4. If the script dies, re-running picks up exactly where it left off

Applied to: `fetch_variations.py` (per variation), `download_images.py` (per image via `url_map.json`), `upload_images.py` (per upload via `upload_map.json`).

---

## Reshape Decisions

### Fields kept
| WooCommerce | Renamed | Reason |
|---|---|---|
| `permalink` | `wpUrl` | 301 redirects from Google-indexed URLs |
| `_links.self.href` | `productUrl` | Debug reference back to WooCommerce API |
| `average_rating` | `averageRating` | Seed data for future review system |
| `review_count` | `reviewCount` | Same |

### Fields dropped
- All currency metadata (`currency_code`, `currency_symbol`, `price_html`) — all USD, frontend hardcodes `$`
- `srcset`, `sizes`, `thumbnail_*` — CDN/storage generates these
- Image `alt` — only 5% populated
- `tags`, `brands` (WooCommerce field) — 4/647 products have tags, 0 have brands
- `_links` block — old WordPress API metadata, useless post-migration
- `stock_status` — always null, `is_in_stock` is the real field
- `is_on_backorder`, `sold_individually`, `is_password_protected` — never true across all products

### `on_sale` computation
```python
sale = product.get("on_sale", False) and sale_price < regular_price
```
Two checks required: owner intent (`on_sale` flag) AND price validation. `sale_price` can exist without the product actively being on sale (may be staged for future use).

### Attribute name mapping
17 raw WooCommerce attribute names mapped to clean keys:
```python
"Choose Color (REQUIRED)" → "color"
"Choose Color"            → "color"
"Filter by Frame Color"   → "frame-color"
"Choose Power"            → "power"
"Choose Transition Type"  → "transition-type"
# ... etc
```
Multiple raw names collapse to the same key (e.g. both color variants → `"color"`). Terms are merged without duplicates.

---

## Architecture — Multi-Brand App

```
[bikershades frontend]  ──┐
[brand2 frontend]       ──┤──→  [Next.js backend + admin dashboard]  ──→  [Supabase]
[brand3 frontend]       ──┘         uses service_role key                  (Postgres + Auth + Storage)
```

- Frontends never touch Supabase directly
- Backend uses `service_role` key — bypasses RLS
- RLS enabled on all tables as a safety net
- `anon` key only needed for client-side Supabase Auth UI — not for data queries

---

## Problems Encountered

### Folder naming: slug vs SKU
First run of `download_images.py` used product slug for folder names. Correct identifier is SKU (what the DB uses as the unique key). Had to stop the job, wipe the `images/` folder entirely, and restart with SKU-based naming. Lesson: decide on the identifier before starting a long download job.

### `url_map.json` mismatch after folder rename
After switching to SKU folders, the existing `url_map.json` had slug-based paths. Would have caused mixed naming. Resolved by wiping everything and starting clean rather than trying to migrate the map.

### Python stdout buffering
`import_products.py` showed no output when running as a background process — Python buffers stdout when not attached to a terminal. Fix: `sys.stdout.reconfigure(line_buffering=True)` at the top of the script.

### `psycopg2` not installed
`supabase` (Python client) was already installed but `psycopg2-binary` was not. Import script failed immediately. Fixed with `pip install psycopg2-binary` before re-running.

### 1 image upload timeout
`BFF2X3AF-0/Smoke-Front-1.jpg` timed out on the first upload run. Because `upload_map.json` checkpoints after every successful upload, re-running the script retried only that one file and succeeded.

---

## Tools Used

| Tool | Purpose |
|---|---|
| Python `requests` | WooCommerce API fetching |
| Python `psycopg2` | Direct Postgres inserts |
| `supabase-py` | Supabase Storage uploads |
| `python-dotenv` | `.env` loading |
| Supabase SQL editor | Schema setup, admin user insertion |
| Supabase Storage | Image hosting (public bucket) |
| Supabase Auth | Admin authentication |
