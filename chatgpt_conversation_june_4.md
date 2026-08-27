# Ecommerce Migration & Rebuild Learnings (June 4, 2026)

*Last Updated: 2026-06-05*

---

# Goal

Rebuild three legacy ecommerce websites:

* ProSportSunglasses
* Sunglass Monster
* BikerShades

using a modern stack and a clean, validated catalog rather than directly migrating years of accumulated WooCommerce data.

The original assumption was:

```text
WooCommerce → Source of Truth
```

The project ultimately revealed a more nuanced reality.

---

# Original Problem

The three storefronts had accumulated years of:

* Manual data entry
* Plugin dependencies
* WooCommerce inconsistencies
* Product duplication
* Inventory synchronization issues
* Store-specific junk products

Examples included:

* USPS return labels
* Rx add-ons
* Face masks
* Warranty products
* Admin-only products
* Private listings
* Draft products

Many products also contained:

* Missing images
* Missing SKUs
* Incorrect stock values
* Parent SKU inconsistencies
* Mixed product families grouped under a single variable product

Because of this, importing WooCommerce directly into a new platform would have transferred years of accumulated catalog problems.

---

# The Migration Strategy

Rather than trusting WooCommerce, the pipeline was designed to answer:

```text
Which records represent real products?
```

and

```text
Which SKUs can be trusted?
```

The pipeline became:

```text
fetch
  ↓
slim
  ↓
validate
  ↓
reshape
  ↓
create-items
  ↓
categorize
  ↓
download
  ↓
upload
  ↓
import
```

Each stage had:

* audit scripts
* verify scripts

so transformations could be validated rather than assumed.

---

# Most Important Architectural Realization

WooCommerce was never actually the final source of truth.

The purpose of WooCommerce became:

```text
Product Discovery
SKU Discovery
Storefront Membership Discovery
```

not:

```text
Canonical Product Data
```

---

# What WooCommerce Became

For each site:

```text
prosportsunglasses.com
bikershades.com
sunglassmonster.com
```

products were fetched separately.

The fact that a SKU appeared on a site became valuable information.

Example:

```text
SKU ABC123
exists on BikerShades
```

That tells us:

```text
This product belongs on BikerShades.
```

even if:

```text
stock
weight
dimensions
pricing
```

may not be trustworthy.

WooCommerce effectively became:

```text
Storefront Membership Database
```

---

# Validation Philosophy

The goal was never:

```text
Fix every product.
```

The goal was:

```text
Detect every suspicious product.
```

If a product looked questionable:

* flag it
* exclude it
* review later

Examples:

* duplicate SKUs
* SKU spaces
* malformed images
* missing prices
* draft products
* orphan variations
* attribute collisions
* mixed SKU bases

Nothing suspicious should silently pass.

---

# SKU Learnings

## Variable Products

WooCommerce parent SKUs often contained:

```text
-0
-0-XT
-0-SM
-COMBOS
-Master-MFB
```

These were frequently WooCommerce-generated identifiers.

Variation SKUs were far more reliable.

Example:

```text
Parent:
FM270X20TX-0

Variations:
FM270X20TX-BK-ECCL-ZMD
FM270X20TX-BK-CL-ZMD
```

The variation SKUs represent actual sellable products.

---

## Simple Products

A major discovery:

```text
-0
```

is not always a WooCommerce suffix.

Examples:

```text
FM270X20TX-0-EC
7EYE-ASPEN-0-HIGHRX
```

are legitimate product SKUs.

Therefore:

```text
Never normalize simple-product SKUs.
```

---

# Base SKU Mismatch Detection

One of the most valuable validation checks.

Rule:

```text
All variations inside a product
must share the same SKU base.
```

Examples:

Valid:

```text
CA541...
CA541...
CA541...
```

Invalid:

```text
CA541...
CAPL541...
```

This exposed:

* mixed product families
* improperly grouped products
* catalog mistakes

before they entered the final dataset.

---

# Combo Product Discovery

Combo products introduced a new pattern:

```text
CPR29X36ST
PR29X36ST
```

The leading:

```text
C
```

indicates combo versions.

The validator was updated to normalize:

```python
CPR29X36ST
PR29X36ST
```

to the same base.

Result:

```text
+12 BikerShades products
+437 variations
```

recovered.

---

# WooCommerce Admin API vs Store API

Critical discovery:

## Store API

Lossy:

```text
is_in_stock only
prices object
limited variation images
```

## Admin API

Complete:

```text
stock_quantity
regular_price
sale_price
variation image
full variation data
```

Future migrations should always start with:

```text
WooCommerce Admin API
```

---

# Veeqo Investigation

Initially believed:

```text
Veeqo = Product Truth
```

because:

* Amazon inventory synced into Veeqo
* Veeqo contained thousands of products
* Veeqo had product exports

Investigation revealed:

## Good

* SKU existence validation
* Active inventory confirmation

## Poor

* Quantity coverage (~8%)
* Cost coverage (~0%)
* Price consistency (~21%)
* ProSport coverage (~10%)

Therefore:

```text
Veeqo ≠ Product Truth
```

---

# Final Understanding of Veeqo

Veeqo became:

```text
SKU Validation Layer
```

Workflow:

```text
Clean WooCommerce SKU
       ↓
Lookup in Veeqo
       ↓
Exists?
       ↓
Yes → confidence boost
No → review
```

Not:

```text
Use Veeqo title
Use Veeqo price
Use Veeqo inventory
```

---

# Amazon Investigation

A major question:

```text
Where do product images live?
```

Discovery:

Amazon images use:

```text
m.media-amazon.com
```

URLs.

This means Amazon hosts its own copies.

Therefore:

```text
WooCommerce image removed
```

does NOT imply:

```text
Amazon image breaks
```

which greatly reduces migration risk.

---

# Site Recovery Discovery

While attempting to obtain WooCommerce API keys:

* Production WordPress site was inaccessible.
* Plugins had become corrupted/missing.
* Staging environments were created.
* Problematic plugin folders were renamed.
* WooCommerce was re-enabled safely.
* WordPress admin access was restored.
* REST API keys were successfully generated.

Unexpectedly:

```text
Fixing the site
```

became part of the API access process.

---

# Pipeline Funnel

Final counts:

| Stage             | ProSport | Monster | BikerShades |
| ----------------- | -------- | ------- | ----------- |
| Fetched           | 171      | 277     | 704         |
| Validate Products | 169      | 267     | 611         |
| Create Items      | 161      | 260     | 565         |
| Veeqo Matched     | 154      | 255     | 562         |

---

# Current Architecture

The final architecture is:

```text
WooCommerce
    ↓
SKU Discovery
Store Membership Discovery
Validation
Catalog Cleanup

    ↓

Canonical Catalog

    ↓

Supabase

    ↓

New Storefronts
```

Veeqo remains:

```text
SKU Validation
```

only.

---

# Biggest Lesson

The biggest lesson was:

```text
Don't ask:
"Which system has the data?"

Ask:
"Which system is trustworthy for each piece of data?"
```

Final answer:

```text
WooCommerce:
    Product membership
    Product content
    Images
    Prices
    Variations

Veeqo:
    SKU confirmation

Amazon:
    Reliable hosted images
```

The project evolved from:

```text
Migrate WooCommerce
```

into:

```text
Build a catalog qualification system
```

that can confidently determine which products are real, valid, and safe to import into a modern ecommerce platform.

