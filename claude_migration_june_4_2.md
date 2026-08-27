Pipeline Shapes & Validation (June 4, 2026)

From WooCommerce admin REST API to final create-items output

1. Slim Product Shape

Source: data/{slug}/slim-products/products.json

{
  "id":                704,
  "name":              "Dynamo Combos [Fits LG-XL Heads]",
  "slug":              "dynamo-combos-lg-xl",
  "permalink":         "https://bikershades.com/shop/...",
  "type":              "variable",
  "status":            "publish",
  "description":       "<h1>...</h1> (raw HTML)",
  "short_description": "<ul><li>...</li></ul> (raw HTML)",
  "sku":               "CFM65X07-0",
  "price":             "23.99",
  "regular_price":     "23.99",
  "sale_price":        "",
  "on_sale":           false,
  "featured":          false,
  "total_sales":       12,
  "stock_quantity":    null,
  "stock_status":      "instock",
  "weight":            "",
  "dimensions": {
    "length": "6",
    "width":  "4",
    "height": "4"
  },
  "categories": [
    { "name": "See All Combo Sets", "slug": "see-all-combo-sets" }
  ],
  "images": [
    { "src": "https://bikershades.com/wp-content/uploads/2020/12/Dynamo.jpg", "name": "Dynamo" }
  ],
  "attributes": [
    { "name": "Choose Color (REQUIRED)", "options": ["Black Smoke", "Clear"] }
  ],
  "variations": [67801, 67802, 67803]
}

Notes:

price, regular_price, sale_price are top-level strings (not inside a prices{} object — that was the old Store API)

stock_quantity is null for variable products where manage_stock=false at the parent level

weight and dimensions values are strings (or empty string "")

variations is a list of integer IDs for variable products; empty for simple

images is a list — all product gallery images; name is the WordPress attachment title

attributes[].options — all possible values across all variations; can be empty [] (filled by variations at create-items)

description and short_description contain raw HTML

2. Slim Variation Shape

Source: data/{slug}/slim-variations/variations.json

The file is an array of per-product entries, each containing the product's variations:

[
  {
    "product_id": 704,
    "variations": [ ... ]
  }
]

Individual variation:

{
  "id":             67771,
  "parent_id":      67767,
  "name":           "White Green Smoke",
  "type":           "variation",
  "status":         "publish",
  "description":    "<p>Topspin w/ White Frame and Green Smoke Lens</p>",
  "permalink":      "https://bikershades.com/shop/.../?attribute_pa_choose-color=white-green-smoke",
  "sku":            "SP47X96BS-WH-GRSM",
  "price":          "19.99",
  "regular_price":  "25.99",
  "sale_price":     "19.99",
  "on_sale":        true,
  "stock_quantity": 10,
  "stock_status":   "instock",
  "weight":         "2",
  "dimensions": {
    "length": "6",
    "width":  "3",
    "height": "3"
  },
  "image": {
    "src":  "https://bikershades.com/wp-content/uploads/.../Front.jpg",
    "name": "Backspin Glossy Black Green Smoke Front"
  },
  "attributes": [
    { "name": "Choose Color (REQUIRED)", "option": "White Green Smoke" }
  ]
}

Key differences from product:

image is a singular object (not an array) — the variation's featured image

image can be null — BikerShades photochromic/transition lens variations often have no image in WooCommerce

attributes[].option is a single string (the chosen term), not an array of options

No categories, featured, total_sales, or short_description

stock_quantity is an integer (real quantity from WooCommerce), not null (unlike parent variable products)

3. Validation — Products (validate_products.py)

Input: slim-products/products.jsonOutput: validate-products/products.json (passed) + flagged.json

Checks performed (per product)

Field

Rule

id

Number, unique across all products

name

Non-empty string

slug

Non-empty string

permalink

Non-empty string

type

"variable" or "simple"

type vs variations

variable must have variation IDs; simple must have none

status

Must be "publish"

description

String or null

short_description

String or null

sku

Non-empty string, no spaces, unique across all products

price

Non-empty string, valid number > 0

regular_price

Null/empty OK; if present must be valid number

sale_price

Null/empty OK; if present must be valid number, and < regular_price

on_sale

Boolean

on_sale consistency (simple only)

on_sale=true requires a sale_price; sale_price set requires on_sale=true

price == regular_price (simple, no sale)

When no sale_price and regular_price present, price must equal regular_price

featured

Boolean

total_sales

Number or null

stock_quantity

Number or null

stock_status

"instock" or "outofstock"

weight

Null/empty OK; if present must be valid number

dimensions.{length,width,height}

Each: null/empty OK; if present must be valid number

categories

Each entry: non-empty name string and non-empty slug string

images

At least one required; each: src starts with "http", name non-empty string

attributes

Each: non-empty name; each option non-empty string

Attribute name collision

Simulates clean_attr_name() — flags if two raw attribute names would map to the same cleaned name

variations

Each ID must be a number

What passes through unflagged:

Empty options[] arrays on attributes — filled by create_items from variation data

Variable products' on_sale consistency — unreliable at product level; checked at variation level

WooCommerce slugs that don't match slugify(name) — WC generates slugs with its own rules

4. Validation — Variations (validate_variations.py)

Input: slim-variations/variations.json + validate-products/products.jsonOutput: validate-variations/variations.json (passed, same structure) + flagged.json

Checks performed (per variation)

Field

Rule

product_id

Number; must exist in the validated products set

id

Number, unique across all variations

parent_id

Number; must match the entry's product_id

name

Non-empty string

type

Must be "variation"

status

Must be "publish"

description

String or null

permalink

Non-empty string

sku

Non-empty string, no spaces, unique across all variations

price

Non-empty string, valid number > 0

regular_price

Null/empty OK; if present must be valid number

sale_price

Null/empty OK; if present must be valid number, and < regular_price

on_sale

Boolean; must be consistent with sale_price (both directions)

price == regular_price (no sale)

When no sale_price and regular_price present, price must equal regular_price

stock_quantity

Number or null

stock_status

"instock" or "outofstock"

weight

Null/empty OK; if present must be valid number

dimensions.{length,width,height}

Each: null/empty OK; if present must be valid number

image

Null is OK — only flagged if present but malformed (missing src, non-http src, empty name)

attributes

Each: non-empty name, non-empty option

Attribute name collision

Same clean_attr_name() simulation as product validation

Cross-product check (per product group)

Check

Rule

Base SKU consistency

All variations within a product must share the same pre-dash SKU segment (e.g., all start with CA541 or all start with CAPL541 — not mixed). Flags the entire group if mismatched.

What gets excluded:

Variations whose product_id is not in the validated products set

Variations with status != "publish" (draft, private, etc.)

Variations with SKU spaces, duplicate SKUs, missing price, malformed images

Variations in products with base SKU mismatch (entire product's variations excluded together)

5. Reshape — Products (reshape_products.py)

Input: validate-products/products.jsonOutput: reshape-products/products.json

Transformations

Input field

Output field

Transformation

id

id

unchanged

name

name

unchanged

slug

slug

unchanged

permalink

oldUrl

renamed

type

type

unchanged

description (HTML)

description

HTML tags stripped to plain text; HTML entities unescaped

description (HTML)

descriptionImages

<img src> URLs extracted → [{src, name, sortOrder}] (1-indexed)

short_description (HTML)

summary

<li> text items extracted → [string, ...]; empty [] if none

sku

sku

unchanged

prices (variable)

minPriceCents, maxPriceCents, salePriceCents

all null — deferred to create_items

prices (simple)

minPriceCents, maxPriceCents

regular_price (or price) × 100, rounded to int

sale_price (simple)

salePriceCents

× 100 if present

on_sale

sale

unchanged (bool)

featured

featured

unchanged

total_sales

totalSales

0 if null/missing

stock_quantity + stock_status (variable)

stock

always null — variation-level is source of truth

stock_quantity + stock_status (simple)

stock

stock_quantity if present; else 1 if instock, 0 if not

weight (variable)

weight

always null

weight (simple)

weight

string → float; null if empty

—

weightUnit

always null

dimensions (variable)

dimensions.{length,width,height}

all null

dimensions (simple)

dimensions

strings → floats; all-or-nothing: if only some fields present, all three set to null with [INFO] print

—

dimensionUnit

always null

categories

categories

unchanged [{name, slug}]

images

images

sortOrder field added (1-indexed); [{src, name, sortOrder}]

attributes

attributes

name cleaned via clean_attr_name(); Power options normalized to "1.50" format

variations

variations

unchanged (list of int IDs)

clean_attr_name() rules (applied in order)

Check ATTR_OVERRIDES[brand] dict — if matched, return override directly

Strip  for Filter suffix (case-insensitive)

Strip Choose  prefix (case-insensitive)

Strip Filter by  prefix (case-insensitive)

Strip parentheticals like  (REQUIRED) (case-insensitive)

6. Reshape — Variations (reshape_variations.py)

Input: validate-variations/variations.jsonOutput: reshape-variations/variations.json (same [{product_id, variations:[]}] structure)

Transformations

Input field

Output field

Transformation

id

id

unchanged

parent_id

parent_id

unchanged

name

name

unchanged

type

type

unchanged

permalink

oldUrl

renamed

description (HTML)

description

HTML stripped to plain text

sku

sku

unchanged

regular_price

regularPriceCents

string × 100 → int; null if empty

sale_price

salePriceCents

string × 100 → int; null if empty

on_sale

sale

unchanged (bool)

stock_quantity + stock_status

stock

stock_quantity if present; else 1 if instock, 0 if not

weight

weight

string → float; null if empty

—

weightUnit

always null

dimensions

dimensions

all-or-nothing: strings → floats; if partial fill, all three → null with [INFO]

—

dimensionUnit

always null

image (singular object or null)

images

null → []; object → [{src, name, sortOrder: 1}]

attributes

attributes

name cleaned via clean_attr_name(); Power options normalized

Key reshape behavior:

image (singular) becomes images[] — an array of 0 or 1 entries

Null image → empty images[], not an error

No price field in output — only regularPriceCents and salePriceCents (int cents)

7. Create Items (create_items.py)

Input: reshape-products/products.json + reshape-variations/variations.jsonOutput: create-items/items.json (passed) + flagged.json

Phase 1: Embed variations into products

Each product's variations list (currently integers) is replaced by the matching variation objects from reshape-variations. Only variations where id is in the product's expected list and parent_id matches are embedded. Unmatched IDs stay as integers and get flagged in Phase 2.

Phase 2: Per-item checks and mutations (variable products only)

Check

Action

Unreplaced variation IDs (integers remain)

Flag

No filled variations after replacement

Flag; variations set to []

Variation attribute name not in parent attributes

Flag

Variation attribute option not in parent options

Option added to parent's attribute list

Attribute prune

Options no variation uses are removed; empty attributes dropped

Variation image dedup

Any variation image whose src matches a parent product image is removed from variation's images[]

minPriceCents / maxPriceCents

Computed as min/max of all variation regularPriceCents

sale

Recomputed from variations: true if any variation has sale=true; WC product-level value overridden

Phase 3: Global SKU uniqueness (all products + their variations)

Scans all product SKUs and all variation SKUs together. If a SKU appears twice (product vs product, variation vs variation, or product vs variation), both items are flagged with a duplicate SKU error.

Final item shape (variable product)

{
  "id":               704,
  "name":             "Dynamo Combos [Fits LG-XL Heads]",
  "slug":             "dynamo-combos-lg-xl",
  "oldUrl":           "https://bikershades.com/shop/...",
  "type":             "variable",
  "description":      "The Dynamo motorcycle sunglasses is a classic...",
  "descriptionImages": [{ "src": "...", "name": "...", "sortOrder": 1 }],
  "summary":          ["Fits Large to Xlarge heads", "ANSI Z.87 safety rated"],
  "sku":              "CFM65X07-0",
  "minPriceCents":    1999,
  "maxPriceCents":    2999,
  "salePriceCents":   null,
  "sale":             false,
  "featured":         false,
  "totalSales":       12,
  "stock":            null,
  "weight":           null,
  "weightUnit":       null,
  "dimensions":       { "length": null, "width": null, "height": null },
  "dimensionUnit":    null,
  "categories":       [{ "name": "See All Combo Sets", "slug": "see-all-combo-sets" }],
  "images":           [{ "src": "...", "name": "...", "sortOrder": 1 }],
  "attributes":       [{ "name": "Color", "options": ["Black Smoke", "Clear"] }],
  "variations": [
    {
      "id":               67771,
      "parent_id":        704,
      "name":             "Black Smoke",
      "type":             "variation",
      "oldUrl":           "https://bikershades.com/shop/.../?attribute_pa_choose-color=black-smoke",
      "description":      "Dynamo with black smoke lens",
      "sku":              "CFM65X07-BK-SM",
      "regularPriceCents": 2399,
      "salePriceCents":   1999,
      "sale":             true,
      "stock":            8,
      "weight":           2.0,
      "weightUnit":       null,
      "dimensions":       { "length": 6.0, "width": 4.0, "height": 4.0 },
      "dimensionUnit":    null,
      "images":           [{ "src": "...", "name": "...", "sortOrder": 1 }],
      "attributes":       [{ "name": "Color", "option": "Black Smoke" }]
    }
  ]
}

For simple products: variations: [], stock is an integer, weight/dimensions are populated from the product level, minPriceCents == maxPriceCents.

Summary: what gets excluded at each stage

Stage

Excluded

Reason

validate-products

status != publish

Draft, private, trash products

validate-products

empty/missing SKU

Can't be imported without a unique identifier

validate-products

SKU with spaces

WooCommerce junk data

validate-products

missing images

Can't be displayed in store

validate-products

attribute name collision

Two attributes would merge to same name after cleaning

validate-variations

parent_id not in validated products

Orphan variations

validate-variations

status != publish

Draft/private variations

validate-variations

missing price

Can't be sold

validate-variations

SKU spaces or duplicates

Data integrity

validate-variations

malformed image

Image present but broken URL

validate-variations

base SKU mismatch

Entire product group — variations belong to different physical products mixed in one WC product

create-items

no filled variations (variable)

All variations excluded at validate; no sellable product

create-items

duplicate SKU (global)

Product or variation SKU collides with another item
