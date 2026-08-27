# Bikershades WooCommerce Product Migration Guide (May 27, 2026)

This guide walks through how to migrate product data from the current WordPress/WooCommerce site into your own app/database.

The basic flow is:

```txt
WooCommerce Store API
        ↓
products.json
        ↓
download images locally
        ↓
reshape/normalize product data
        ↓
upload images to your storage
        ↓
insert clean products into your database
```

---

## 0. What you are migrating

The WooCommerce product endpoint returns an array of product objects:

```txt
https://bikershades.com/wp-json/wc/store/products?per_page=100&page=1
```

Each product can include:

- `id`
- `name`
- `slug`
- `description`
- `short_description`
- `sku`
- `prices`
- `images`
- `categories`
- `attributes`
- `variations`
- `is_in_stock`
- `stock_availability`
- `weight`
- `dimensions`
- `permalink`

The endpoint is paginated, so one request is not enough. You need to keep fetching pages until the response returns an empty array.

---

## 1. Create a migration folder

In your project, create a separate folder:

```bash
mkdir migration
cd migration
```

Create this structure:

```txt
migration/
  fetch_products.py
  download_images.py
  reshape_products.py
  products.json
  clean_products.json
  images/
```

`products.json` will be your raw backup from WooCommerce.

`clean_products.json` will be your reshaped version for your own app/database.

---

## 2. Install Python dependencies

You only need `requests` for now:

```bash
pip install requests
```

If you use a virtual environment:

```bash
python3 -m venv venv
source venv/bin/activate
pip install requests
```

---

## 3. Fetch all products from WooCommerce

Create `fetch_products.py`:

```python
import json
import time
import requests

BASE_URL = "https://bikershades.com/wp-json/wc/store/products"
PER_PAGE = 100

all_products = []
page = 1

while True:
    url = f"{BASE_URL}?per_page={PER_PAGE}&page={page}"

    print(f"Fetching page {page}...")
    response = requests.get(url, timeout=30)

    if response.status_code != 200:
        print(f"Stopped. HTTP {response.status_code}")
        print(response.text[:500])
        break

    products = response.json()

    if not isinstance(products, list):
        print("Stopped. Response was not a list.")
        print(products)
        break

    if len(products) == 0:
        print("No more products.")
        break

    all_products.extend(products)
    print(f"Fetched {len(products)} products from page {page}.")

    page += 1
    time.sleep(0.5)

with open("products.json", "w", encoding="utf-8") as file:
    json.dump(all_products, file, indent=2, ensure_ascii=False)

print(f"Saved {len(all_products)} total products to products.json")
```

Run it:

```bash
python fetch_products.py
```

After this, you should have:

```txt
products.json
```

This file is your raw backup of the WooCommerce product catalog.

---

## 4. Download product images locally

Create `download_images.py`:

```python
import os
import json
import requests
from urllib.parse import urlparse

IMAGE_DIR = "images"

os.makedirs(IMAGE_DIR, exist_ok=True)

with open("products.json", "r", encoding="utf-8") as file:
    products = json.load(file)

downloaded = set()

for product in products:
    product_id = product.get("id")
    product_name = product.get("name", "unknown-product")

    for image in product.get("images", []):
        image_url = image.get("src")

        if not image_url:
            continue

        if image_url in downloaded:
            continue

        try:
            response = requests.get(image_url, timeout=30)

            if response.status_code != 200:
                print(f"Failed {response.status_code}: {image_url}")
                continue

            parsed_url = urlparse(image_url)
            original_filename = os.path.basename(parsed_url.path)

            if not original_filename:
                print(f"Skipping image with no filename: {image_url}")
                continue

            # Prefix with product id to avoid filename collisions.
            filename = f"{product_id}-{original_filename}"
            filepath = os.path.join(IMAGE_DIR, filename)

            with open(filepath, "wb") as img_file:
                img_file.write(response.content)

            downloaded.add(image_url)
            print(f"Downloaded: {filename} from {product_name}")

        except Exception as error:
            print(f"Error downloading {image_url}")
            print(error)

print(f"Downloaded {len(downloaded)} unique images.")
```

Run it:

```bash
python download_images.py
```

This creates:

```txt
images/
  67767-Backspin-Glossy-Black-Green-Smoke-Front.jpg
  67767-Backspin-Glossy-Black-Green-Smoke-Angle.jpg
  ...
```

---

## 5. Why download/rehost images?

Do not permanently use old WordPress URLs like:

```txt
https://bikershades.com/wp-content/uploads/...
```

because your new site would depend on the old site staying alive.

If the WordPress site is deleted, changed, moved, blocks hotlinking, or expires, your new site’s images break.

Better flow:

```txt
old WordPress image URL
        ↓
download image locally
        ↓
upload to your own storage
        ↓
save your new image URL in your database
```

Storage options:

- Supabase Storage
- AWS S3
- Cloudinary
- Shopify files
- Vercel Blob

---

## 6. Reshape the raw WooCommerce JSON

The WooCommerce JSON is their schema. You probably want your own simpler schema.

Create `reshape_products.py`:

```python
import json
import re
from html import unescape

def strip_html(html):
    if not html:
        return ""

    text = re.sub(r"<[^>]+>", " ", html)
    text = unescape(text)
    text = re.sub(r"\s+", " ", text)

    return text.strip()

def cents_to_dollars(cents_string):
    if cents_string is None or cents_string == "":
        return None

    return int(cents_string) / 100

with open("products.json", "r", encoding="utf-8") as file:
    products = json.load(file)

clean_products = []

for product in products:
    prices = product.get("prices") or {}

    clean_product = {
        "oldWooCommerceId": product.get("id"),
        "name": product.get("name"),
        "slug": product.get("slug"),
        "sku": product.get("sku") or None,
        "type": product.get("type"),
        "oldUrl": product.get("permalink"),

        "descriptionHtml": product.get("description") or "",
        "descriptionText": strip_html(product.get("description") or ""),
        "shortDescriptionText": strip_html(product.get("short_description") or ""),

        "priceInCents": int(prices.get("price") or 0),
        "regularPriceInCents": int(prices.get("regular_price") or 0),
        "salePriceInCents": int(prices.get("sale_price") or 0),
        "currency": prices.get("currency_code") or "USD",
        "onSale": product.get("on_sale", False),

        "isInStock": product.get("is_in_stock", False),
        "stockText": (product.get("stock_availability") or {}).get("text", ""),

        "categories": [
            {
                "oldId": category.get("id"),
                "name": category.get("name"),
                "slug": category.get("slug"),
                "oldUrl": category.get("link"),
            }
            for category in product.get("categories", [])
        ],

        "images": [
            {
                "oldImageId": image.get("id"),
                "oldUrl": image.get("src"),
                "thumbnailUrl": image.get("thumbnail"),
                "alt": image.get("alt") or "",
                "name": image.get("name") or "",
            }
            for image in product.get("images", [])
        ],

        "attributes": product.get("attributes", []),
        "variations": product.get("variations", []),

        "weight": product.get("weight") or None,
        "dimensions": product.get("dimensions") or {},
    }

    clean_products.append(clean_product)

with open("clean_products.json", "w", encoding="utf-8") as file:
    json.dump(clean_products, file, indent=2, ensure_ascii=False)

print(f"Saved {len(clean_products)} clean products to clean_products.json")
```

Run it:

```bash
python reshape_products.py
```

Now you have:

```txt
clean_products.json
```

This is easier to import into your own database.

---

## 7. Example database schema

A clean relational schema could look like this:

```sql
CREATE TABLE products (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  old_woocommerce_id INTEGER UNIQUE,
  name TEXT NOT NULL,
  slug TEXT UNIQUE NOT NULL,
  sku TEXT,
  type TEXT,
  description_html TEXT,
  description_text TEXT,
  price_in_cents INTEGER NOT NULL DEFAULT 0,
  regular_price_in_cents INTEGER NOT NULL DEFAULT 0,
  sale_price_in_cents INTEGER NOT NULL DEFAULT 0,
  currency TEXT NOT NULL DEFAULT 'USD',
  on_sale BOOLEAN NOT NULL DEFAULT FALSE,
  is_in_stock BOOLEAN NOT NULL DEFAULT FALSE,
  stock_text TEXT,
  old_url TEXT,
  created_at TIMESTAMP DEFAULT now(),
  updated_at TIMESTAMP DEFAULT now()
);

CREATE TABLE product_images (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  product_id UUID REFERENCES products(id) ON DELETE CASCADE,
  old_url TEXT,
  new_url TEXT,
  alt TEXT,
  sort_order INTEGER DEFAULT 0
);

CREATE TABLE categories (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  old_woocommerce_id INTEGER UNIQUE,
  name TEXT NOT NULL,
  slug TEXT UNIQUE NOT NULL
);

CREATE TABLE product_categories (
  product_id UUID REFERENCES products(id) ON DELETE CASCADE,
  category_id UUID REFERENCES categories(id) ON DELETE CASCADE,
  PRIMARY KEY (product_id, category_id)
);

CREATE TABLE product_variants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  product_id UUID REFERENCES products(id) ON DELETE CASCADE,
  old_variation_id INTEGER UNIQUE,
  attributes JSONB,
  created_at TIMESTAMP DEFAULT now()
);
```

This is only a starting point. You should adjust based on your app.

---

## 8. Important choice: simple products vs variable products

WooCommerce has different product types.

Simple product:

```json
{
  "type": "simple"
}
```

Variable product:

```json
{
  "type": "variable",
  "attributes": [...],
  "variations": [...]
}
```

A variable product is something like:

```txt
Sunglasses
  - Color: Black
  - Color: Tortoise
  - Power: 1.50
  - Power: 2.00
```

For variable products, the main endpoint may only give variation IDs and selected attributes. If you need exact variant-level price, stock, or images, fetch each individual product or variation endpoint separately.

Start by migrating the parent products first. Then handle variants as a second pass.

---

## 9. Upload images to your own storage

At this point you have local files in:

```txt
images/
```

Next you upload them to your storage provider.

For example:

```txt
images/67767-Backspin-Front.jpg
        ↓
Supabase Storage / S3 / Cloudinary
        ↓
https://your-storage.com/products/67767-Backspin-Front.jpg
```

Then update your product image records:

```txt
old_url = old WordPress URL
new_url = your hosted URL
```

Keep `old_url` for debugging/mapping.

---

## 10. Import into your DB

At this point you will have:

```txt
clean_products.json
images/
```

Then write an import script for your database.

The exact code depends on what you use:

- Supabase Python client
- psycopg2
- Prisma
- Drizzle
- SQLAlchemy

The import process should be:

```txt
for each clean product:
  insert product
  insert categories if missing
  connect product to categories
  insert image rows
  insert variants if any
```

---

## 11. Recommended order

Do this in stages:

### Stage 1: Raw backup

```bash
python fetch_products.py
```

Confirm:

```txt
products.json exists
```

### Stage 2: Images

```bash
python download_images.py
```

Confirm:

```txt
images/ contains product images
```

### Stage 3: Clean JSON

```bash
python reshape_products.py
```

Confirm:

```txt
clean_products.json exists
```

### Stage 4: Design DB schema

Create:

```txt
products
product_images
categories
product_categories
product_variants
```

### Stage 5: Upload images

Upload local images to your own storage.

### Stage 6: Import DB records

Insert clean product records into your DB.

### Stage 7: Build frontend

Your new frontend should read from your DB, not directly from WordPress.

---

## 12. Full migration checklist

- [ ] Create `migration/` folder
- [ ] Install Python requests
- [ ] Run `fetch_products.py`
- [ ] Confirm `products.json` has all paginated products
- [ ] Run `download_images.py`
- [ ] Confirm images downloaded locally
- [ ] Run `reshape_products.py`
- [ ] Confirm `clean_products.json`
- [ ] Decide your database schema
- [ ] Upload images to your own storage
- [ ] Save new image URLs
- [ ] Insert products into DB
- [ ] Insert product images into DB
- [ ] Insert categories into DB
- [ ] Insert variants into DB
- [ ] Build product listing page
- [ ] Build product detail page
- [ ] Compare new site against old WooCommerce pages

---

## 13. Key idea

The migration is not one big magical step.

It is a pipeline:

```txt
extract → backup → clean → rehost assets → insert → render
```

Do not skip the backup step. Keep the original `products.json` untouched so you can always rerun later steps without refetching everything.
