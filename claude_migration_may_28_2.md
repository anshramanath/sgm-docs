# Bikershades Migration — What We Built & What I Learned (May 28, 2026)

---

## What We Built

### `fetch_products.py`
Fetches all products from the WooCommerce Store API using pagination and saves them to `products.json`. Stops when the API returns an empty page.

**Result:** 647 products across 7 pages.

### `fetch_variations.py`
Fetches full variation data for every variation ID found in `products.json`. Saves to `variations.json` keyed by parent product ID.

**Result:** 6,064 variations across 486 variable products.

- Checkpoints after every individual variation — crash-safe at any point
- On resume, checks which variation IDs are already saved per parent and fetches only missing ones

### `reshape_products.py`
Cleans and normalizes the raw WooCommerce data into a lean schema, merging variation data from `variations.json` into each product.

**Result:** `clean_products.json` — 639 products (8 zero-price internal products filtered out).

---

## What I Learned

### APIs & Pagination
- `per_page` limits items per request to protect server performance — unrelated to how many the website shows visually
- Same products whether fetched as `per_page=100` or `per_page=10` — just split across more requests
- Paginate by incrementing `page` until the API returns an empty array

### WooCommerce Product Types
- **Simple** — one SKU, no variants
- **Variable** — has color/option variants, each with their own SKU, images, price, and stock
- Pagination only returns parent products — variations excluded by design
- Variations fetched via `/products/{variation_id}`, same endpoint as a regular product
- Whether a product is variable is determined by whether it has variations — not by the `type` field

### Slugs & URLs
- Slug — URL-safe version of a name, lowercase with dashes
- `permalink` → `wpUrl` — kept for 301 redirects and comparing against the live site
- `_links.self.href` → `productUrl` — kept as a single debug reference back to the WooCommerce API

### 301 Redirects
- Permanent redirect — tells browsers and Google the page has moved forever
- Google transfers SEO ranking from the old URL to the new one over time
- Set up in Next.js via `next.config.js`, `vercel.json`, or Nginx

### SKUs
- Stock Keeping Unit — unique identifier per product variant for inventory tracking
- Made up by the store owner, used for warehouse management and reordering

### Variations & Attributes
- Variations are a full cartesian product — 4 colors × 11 transition types = 44 variations
- Each variation has a `parent` field and a `variation` string (e.g. `"Choose Color: Black, Choose Power: 1.50"`)
- `has_variations` on attributes indicates whether that attribute drives variation splits vs being a label
- 38 variations have empty `variation` strings — recovered from the `attributes` array on the variation stub in `products.json`

### Prices
- Stored as strings in cents — `"1999"` = $19.99
- `sale` requires two checks: `on_sale == true AND sale_price < regular_price`
- All 647 products are USD only — entire currency block dropped
- 8 products have `regular_price = 0` — internal/service products, filtered out

### Stock
- `is_on_backorder` never true across all products and variations — dropped
- Stock kept binary (`inStock`) — exact counts not surfaced to customers

### Images
- Only the original `src` kept — CDN/storage handles resizing
- `alt` dropped — only 5% of images have it
- Descriptions contain embedded `<img>` tags — URLs extracted into `descriptionImgs[]` for download/rehost
- 587 products have description images (990 total URLs), 4 variations have them

### Data Decisions
- `summary` parsed from `<li>` tags into string array, plain text fallback for 56 products that use `<p>` tags
- `description` stripped to plain text for both products and variations
- `averageRating` normalized to float
- `categories` as flat slug array — for filtering only
- `attributes` as flat term slug array — all kept regardless of `has_variations`
- `tags` (4/647) and `brands` (0/647) dropped
- Variation `summary`, `averageRating`, `reviewCount` dropped — always empty/zero at variation level

### Clean Shape

**Product:**
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
  "regularPriceInCents": 2599,
  "salePriceInCents": 1999,
  "averageRating": 4.5,
  "reviewCount": 12,
  "inStock": true,
  "categories": ["prosport-sunglasses", "smoke-grey"],
  "images": [{ "src": "https://bikershades.com/wp-content/...", "name": "Backspin Front" }],
  "attributes": ["blue-green-smoke", "glossy-black-green-smoke"],
  "variations": [
    {
      "attributes": ["blue-green-smoke"],
      "variation": {
        "name": "Backspin Sport Sunglasses",
        "slug": "backspin-sport-sunglasses-blue-green-smoke",
        "sku": "SP47X96BS-BL-GRSM",
        "variation": "Choose Color (REQUIRED): Blue Green Smoke",
        "wpUrl": "https://bikershades.com/shop/.../?attribute=blue-green-smoke",
        "productUrl": "https://bikershades.com/wp-json/wc/store/products/67768",
        "description": "Backspin w/ Blue Frame and Green Smoke Lens",
        "descriptionImgs": [],
        "sale": true,
        "regularPriceInCents": 2599,
        "salePriceInCents": 1999,
        "inStock": true,
        "images": [{ "src": "https://bikershades.com/wp-content/...", "name": "Backspin Blue" }],
        "weight": 2,
        "weightUnit": "oz",
        "dimensions": { "length": 6, "width": 3, "height": 3 },
        "dimensionUnit": "in"
      }
    }
  ],
  "weight": 2,
  "weightUnit": "oz",
  "dimensions": { "length": 6, "width": 3, "height": 3 },
  "dimensionUnit": "in"
}
```
