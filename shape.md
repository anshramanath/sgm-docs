# Reshaped Item Structure (June 1, 2026)
_Last updated: 2026-06-01_

Documents the JSON shape produced by `reshape_items.py` and every decision that went into it.

---

## Full Product Shape

```json
{
  "name": "Backspin Sport Sunglasses",
  "slug": "backspin-sport-sunglasses",
  "sku": "SP47X96BS-0",
  "wpUrl": "https://bikershades.com/shop/...",
  "productUrl": "https://bikershades.com/wp-json/wc/store/products/67767",
  "description": "Capture the essence of athletic performance...",
  "descriptionImgs": ["https://bikershades.com/wp-content/uploads/diagram.jpg"],
  "summary": ["Fits Large to X-Large heads", "Impact resistant lenses"],
  "sale": true,
  "minPriceCents": 1999,
  "maxPriceCents": 3499,
  "salePriceCents": null,
  "stock": 1,
  "categories": ["https://bikershades.com/product-category/sunglasses/brands/prosport/"],
  "images": [
    { "src": "https://bikershades.com/wp-content/uploads/...", "name": "Backspin Front" }
  ],
  "attributes": [
    {
      "name": "color",
      "terms": [
        { "name": "Blue Green Smoke", "slug": "blue-green-smoke" },
        { "name": "Black Clear", "slug": "black-clear" }
      ]
    }
  ],
  "variations": [
    {
      "attribute": ["blue-green-smoke"],
      "variation": {
        "slug": "backspin-sport-sunglasses-blue-green-smoke",
        "sku": "SP47X96BS-BG",
        "wpUrl": "https://bikershades.com/?p=65388",
        "productUrl": "https://bikershades.com/wp-json/wc/store/products/65388",
        "description": "",
        "sale": true,
        "regularPriceCents": 1999,
        "salePriceCents": 1599,
        "stock": 1,
        "images": [],
        "weight": null,
        "weightUnit": null,
        "dimensions": { "length": null, "width": null, "height": null },
        "dimensionUnit": null
      }
    }
  ],
  "weight": 2.0,
  "weightUnit": "oz",
  "dimensions": { "length": 6.0, "width": 3.0, "height": 3.0 },
  "dimensionUnit": "in"
}
```

---

## Field-by-Field Decisions

### `name`

HTML entities decoded (`&amp;` → `&`) and any stray HTML tags stripped. WooCommerce names sometimes arrive with encoded characters or formatting artifacts from the CMS.

---

### `slug`

Taken directly from WooCommerce — already URL-safe and unique. Used for frontend routing (`/products/:slug`).

---

### `sku`

`null` if missing or blank. Invalid SKUs (null or containing spaces) go to `invalid_sku_items` bucket — not imported. Global uniqueness enforced via post-processing pass in reshape: product SKU and all variation SKUs are pooled into a global set. If any collide, the whole product goes to `duplicate_sku_items`.

**Why global uniqueness matters:** the DB has a `UNIQUE` constraint on `variations.sku`. If two products share a variation SKU, the import fails. Enforcing this at reshape time surfaces the problem before import rather than crashing mid-run.

---

### `wpUrl`

WooCommerce `permalink` — the old bikershades.com product URL. Kept for 301 redirects from Google-indexed pages and for cross-referencing against the live site during review.

---

### `productUrl`

`_links.self[0].href` — direct URL to this product's WooCommerce API endpoint. Kept as a single debug reference. Makes it trivial to inspect the original raw data for any product without digging through `products.json`.

Both `wpUrl` and `productUrl` are dropped from `_links` bulk import — only the two useful ones are preserved.

---

### `description`

All HTML stripped to plain text. Embedded `<img>` URLs extracted separately into `descriptionImgs` before stripping. Two jobs, two fields — the rendered text goes into `description`, the image URLs go through the download/upload pipeline.

WooCommerce's `price_html` (pre-rendered price HTML) and `short_description` become `summary` (see below) — `description` is only the long description.

---

### `descriptionImgs`

Array of image URLs extracted from `<img src="...">` tags in the raw description HTML. These images are embedded inline in the description (product diagrams, size charts, etc.) and need to be rehosted — they still point at bikershades.com's server.

They go through the same download/upload pipeline as product and variation images, with their Supabase URLs stored in `description_images` table. **Product-only** — variation descriptions never had unique description images (the 4 that existed were duplicates of the parent's description images, so they're dropped).

---

### `summary`

WooCommerce `short_description`. Parsed from `<li>` tags into a string array — most products use a `<ul>` bullet list. Falls back to a single-item array of stripped plain text if no `<li>` tags are found. Empty array if the field is blank (31.7% of products).

Frontend hides this section when empty.

---

### `sale` / `minPriceCents` / `maxPriceCents` / `salePriceCents`

**Variable products (have variations):**
- `minPriceCents` = min of all variation `regularPriceCents`
- `maxPriceCents` = max of all variation `regularPriceCents`
- `salePriceCents` = `null` always — the price range is based on regular prices; actual sale prices live on the variations
- `sale` = `true` if any variation has `sale = true`

**Simple products (no variations):**
- `minPriceCents = maxPriceCents = regularPrice`
- `salePriceCents` = null when not on sale, value when on sale
- `sale` = `salePriceCents > 0 AND salePriceCents < minPriceCents`

**Why this model:**
WooCommerce's `on_sale` flag is unreliable — it can be true even when `sale_price == regular_price`. The two-check logic (`sale_price > 0 AND sale_price < regular_price`) is the source of truth. For variable products, the product-level sale price is meaningless (each variation has its own price) so it's always null.

All price fields are integers in cents — WooCommerce returns them as strings (`"1999"`), which are cast to `int` in reshape.

---

### `stock`

`1` (in stock) or `0` (out of stock), derived from WooCommerce's `is_in_stock` boolean. The DB column is `int` not `boolean` so the owner can update it to real quantities later. Frontend treats `stock > 0` as available.

`average_rating` and `review_count` are dropped entirely — the new app builds its own review system from scratch.

---

### `categories`

An array of WooCommerce category `link` URLs — the old bikershades.com category page URLs (e.g. `https://bikershades.com/product-category/sunglasses/brands/prosport/`).

The link URL encodes the full hierarchy — `recategorize_items.py` parses it to derive `name`, `slug`, and depth level. The link is the source of truth; no data is stored here beyond the URL.

**Why links and not slugs:** WooCommerce's category objects have both a display name and a slug, but the hierarchy is only recoverable from the link URL. Keeping the link lets `recategorize_items.py` reconstruct the full tree without needing a separate category map.

---

### `images`

Product gallery images — `src` (full-res original) and `name` only. Everything else dropped:
- `alt` — only ~5% populated; not worth carrying empty strings
- `srcset`, `sizes`, `thumbnail`, `thumbnail_srcset`, `thumbnail_sizes` — Supabase Storage / CDN handles responsive delivery; these are stale WooCommerce-generated values

---

### `attributes`

Array of `{ name, terms[] }` objects. `name` is a normalized short key (see mapping below). `terms` is `[{ name, slug }]`.

**Attribute name normalization:** WooCommerce uses verbose display strings as attribute names (`"Choose Color (REQUIRED)"`, `"Filter by Frame Color"`, etc.). These are mapped to short slugs in `ATTRIBUTE_NAME_MAP`:

| WooCommerce name | Normalized key |
|---|---|
| Choose Color (REQUIRED) / Choose Color | `color` |
| Choose Power | `power` |
| Choose Transition Type | `transition-type` |
| Filter by Frame Color / Choose Frame Color (Required) | `frame-color` |
| Choose Style Color | `style-color` |
| Choose Bandana Color | `bandana-color` |
| Filter by Size / Choose Size | `size` |
| Choose 7 Eye Style | `7-eye-style` |
| Filter by Brand | `brand` |
| Filter by Foam Type | `foam-type` |
| Shipping Option | `shipping` |
| Original Sunglass Purchase Price | `original-price` |
| Qty (packs of 3) | `quantity-packs-3` |
| LENS - GENUINE TRANSITIONS | `lens-genuine-transition` |

**Power slug normalization:** WooCommerce stored diopter values inconsistently as `1-00` and `1.00` for the same power. Normalized to decimal (`1.00`) in reshape.

**Term pruning:** After bucketing, unused attribute terms are stripped from each product — only terms that have an actual matching variation are kept. This ensures every color swatch on the frontend has a selectable variation behind it. Products with mismatched variation slugs (variation attribute value not in product's term list) go to `mismatched_attr_items` before pruning.

**Why store on the product:** Attributes are stored as `jsonb` on the product, not only on variations. Listing pages and category pages need to show color swatches and filter options without joining all variation rows. The product-level attributes are the source of truth for UI; the variation `attribute` array is the source of truth for what's actually in stock per option.

---

### `variations`

Array of `{ attribute, variation }` objects.

**`attribute`:** Array of raw slug values from the WooCommerce variation stub — the selected option combo for this variation (e.g. `["black-clear", "1.00"]`). Order matches the parent product's `attributes` array order. Values are slugified and power-normalized.

**`variation`:** Full variation object (see below). Merged from `variations.json` — WooCommerce's main products endpoint only returns variation stubs (ID + attribute slugs). Full data (price, stock, images, weight) comes from a separate per-variation API fetch.

**Variation images deduplication:** During reshape, variation images that duplicate a parent product image are stripped. The parent product gallery already contains these shots — keeping them on the variation would mean downloading and uploading the same file twice and storing two DB rows pointing at the same image. Frontend falls back to the product gallery for any variation without its own images (64.9% of variations).

---

## Full Variation Shape

```json
{
  "slug": "backspin-sport-sunglasses-blue-green-smoke",
  "sku": "SP47X96BS-BG",
  "wpUrl": "https://bikershades.com/?p=65388",
  "productUrl": "https://bikershades.com/wp-json/wc/store/products/65388",
  "description": "",
  "sale": true,
  "regularPriceCents": 1999,
  "salePriceCents": 1599,
  "stock": 1,
  "images": [],
  "weight": null,
  "weightUnit": null,
  "dimensions": { "length": null, "width": null, "height": null },
  "dimensionUnit": null
}
```

Variations have no `name` (implied by the parent's name + selected attribute), no `categories`, no `attributes`, no `descriptionImgs`, no `summary`.

**`salePriceCents` on variations:** Nullable — `null` when `sale=false`, a value (always `< regularPriceCents`) when `sale=true`. The `sale` bool is always the authoritative flag; never infer from `salePriceCents`.

---

### `weight` / `weightUnit` / `dimensions` / `dimensionUnit`

WooCommerce provides two fields: the raw numeric value (`weight`, `dimensions`) and a formatted display string (`formatted_weight`, `formatted_dimensions`). The unit is only in the formatted string (e.g. `"6 oz"`, `"4 × 2 × 1 in"`).

**Decision:** unit is derived from the last word of the formatted string. If the formatted string is missing, `"N/A"`, or the raw value is 0.0/missing, both the value and the unit are set to `null`. Partial dimensions (any of length/width/height missing) null all three dimensions and the unit together — a partial dimension record is useless.

**Why null instead of a fallback:** An incorrect unit is worse than no unit. `null` surfaces cleanly in the admin dashboard for the owner to fill in; a guessed unit silently corrupts data.

Each product and variation is checked independently — a product can have null weight while its variations have valid weights, and vice versa.

---

## What Was Dropped

| WooCommerce field | Reason |
|---|---|
| `id` | DB generates its own IDs |
| `parent` | Redundant — implied by the nesting structure |
| `type` | Inferable from whether `variations` is non-empty |
| `price_html` | Pre-rendered WooCommerce HTML — stale post-migration |
| `price_range` | Derived from variation prices in reshape |
| `on_sale` | Unreliable; two-check logic is used instead |
| `average_rating`, `review_count` | New app builds its own review system |
| `tags`, `brands` | 4/647 products have tags, 0 have brands — effectively unused |
| `srcset`, `sizes`, `thumbnail*` | CDN handles responsive delivery |
| `alt` on images | Only 5% populated |
| `is_purchasable`, `is_password_protected` | Always true/false respectively across the whole catalog |
| `is_on_backorder`, `low_stock_remaining` | Never set — bikershades uses binary in_stock |
| `stock_availability` (text + class) | `is_in_stock` bool is the source of truth |
| `sold_individually` | Always false |
| `has_options`, `has_variations` | Type derived from `variations` array instead |
| `extensions`, `add_to_cart`, `_links` | API metadata for the WooCommerce frontend |
| `currency_*` fields | All products are USD; frontend hardcodes `$` |
| `grouped_products` | Unused |
| Variation `summary` | Always empty at variation level |
| Variation `categories`, `attributes` | Inherited from parent; not stored on variations |
| Variation `descriptionImgs` | The 4 that existed were duplicates of parent description images |

---

## Bucketing Logic

Products are assigned to exactly one bucket. The checks run in priority order:

1. **`zero_price_items`** — `regular_price == 0` at the product level (internal/service products). Caught first, before reshape checks, because zero-price products often have other issues that would cause false positives.
2. **`invalid_sku_items`** — product SKU is null or contains a space.
3. **`zero_price_items`** (second check) — variable products where all variations have `regularPriceCents == 0`.
4. **`missing_attributes_items`** — any variation has a completely empty `attribute` array.
5. **`mismatched_attr_items`** — any variation attribute slug is not present in the parent product's attribute terms.
6. **`no_image_items`** — product has no images.
7. **`duplicate_name_items`** — product name appears in more than one product (across all non-zero-price products).
8. **`shaped_items`** — passes all checks.

**Post-processing SKU dedup:** After the main bucketing loop, a single pass walks every bucket and maintains a global `seen_skus` set. Any product whose product SKU or any variation SKU collides with a previously-seen SKU (internal collision or cross-product collision) is moved to `duplicate_sku_items`. This runs after the loop so zero-price products (which use `continue` in the main loop) are correctly included in the pass.
