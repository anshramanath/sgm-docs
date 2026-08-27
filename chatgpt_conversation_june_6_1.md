# migration_master_learnings.md (June 6, 2026)

*Last Updated: 2026-06-06*

---

# Project Goal

Rebuild the ProSport, Sunglass Monster, and BikerShades storefronts from a reliable inventory source while eliminating years of WooCommerce data quality issues.

The long-term architecture is:

```text
Amazon + eBay
       ↓
     Veeqo
       ↓
   Supabase
       ↓
  Next.js Storefronts
```

Veeqo should become the operational source of truth for active inventory.

The storefronts should consume normalized data from Supabase rather than depending directly on WooCommerce.

---

# Core Discovery

The original assumption was:

```text
WooCommerce
    ↓
 reliable product catalog
```

This turned out to be incorrect.

WooCommerce contains:

* junk products
* abandoned products
* RX service products
* warranty products
* try-before-you-buy products
* duplicate products
* mixed product families
* missing images
* missing SKUs
* inconsistent categorization

However, WooCommerce still contained one extremely valuable thing:

```text
SKUs
```

The SKU system is the most important artifact preserved from the legacy stores.

The migration pipeline became a process of determining:

```text
Which SKUs represent real sellable products?
```

rather than:

```text
Which WooCommerce products should be migrated?
```

---

# Evolution Of The Strategy

## Original Plan

```text
WooCommerce
      ↓
Clean
      ↓
Import
```

---

## Current Plan

```text
WooCommerce
      ↓
Find valid SKUs
      ↓
Match against Veeqo
      ↓
Pull reliable data
      ↓
Import into Supabase
```

WooCommerce is now primarily used as:

```text
SKU discovery
```

instead of:

```text
catalog authority
```

---

# Multi-Brand Pipeline

Three brands:

```text
proSPORT
Sunglass Monster
BikerShades
```

Each brand runs through the same pipeline.

```text
fetch
  ↓
slim
  ↓
validate
  ↓
reshape
  ↓
create_items
```

Data is isolated per brand:

```text
data/prosport/
data/monster/
data/bikershades/
```

Active brand selected via:

```python
BRAND=<slug>
```

---

# WooCommerce Learnings

## Admin API > Store API

Admin API provides:

* stock_quantity
* regular_price
* sale_price
* variation image data
* complete variation information

Store API lost critical information.

Future migrations should use Admin API from day one.

---

## Parent SKUs Cannot Be Trusted

WooCommerce appends dedupe suffixes:

```text
-0
-0-XT
-0-SM
```

These are WooCommerce artifacts.

Parent SKUs should not be treated as canonical identifiers.

---

## Variation SKUs Are The Real Product IDs

Examples:

```text
FM85X11MN-BK-CL-MFB
RE14X881GN-GD-GSM-100-MFS
CPRE5X07ZA-BL-CL-150
```

Variation SKUs contain the actual product identity.

---

## SKU Base Validation

Major discovery:

Products grouped inside WooCommerce often contain variations from entirely different product families.

Example:

```text
CA10X15
CA10X16
```

inside one variable product.

These should not belong together.

Validation was added to detect:

```text
base sku mismatch
```

and prevent corrupted groupings from entering the pipeline.

---

# Major Validation Rules

## Product Validation

Flags:

* missing SKU
* missing images
* draft status
* private status
* pending status
* attribute collisions
* missing slug
* invalid product types

---

## Variation Validation

Flags:

* base SKU mismatch
* duplicate SKU
* missing price
* malformed image
* invalid status

---

## C-Prefix Discovery

Combo products use:

```text
CPR...
```

while standard products use:

```text
PR...
```

These are valid pairings.

Validation now normalizes:

```python
CPR29X36ST
PR29X36ST
```

to the same base.

This recovered dozens of legitimate products.

---

# Brand Quality Findings

## ProSport

Cleanest dataset.

Issues primarily:

* mixed variation groupings
* minor SKU issues

---

## Sunglass Monster

Generally clean.

Issues primarily:

* mixed variation groupings
* occasional missing fields

---

## BikerShades

Largest catalog.

Most problematic dataset.

Contains:

* RX products
* warranty products
* face masks
* shipping labels
* body armour products
* placeholders
* accessories
* admin products

Large portion of validation effort focused on identifying and excluding non-product records.

---

# Veeqo Discoveries

## Most Important Discovery

Veeqo stores:

```text
variants
```

not:

```text
product groups
```

The inventory view is variation-centric.

This creates a grouping problem.

---

## Why Grouping Matters

The migration ultimately needs:

```text
Product
    ↓
Variations
```

structure.

Example:

```text
shoe
 ├─ black
 ├─ white
 └─ gray
```

A product family identifier is needed.

---

## Family SKU Concept

Current preferred design:

```text
FAMILY-OPTION-OPTION
```

Example:

```text
FM270X19DT-BK-ECCL-ZMD
FM270X19DT-BK-YL-ZMD
FM270X19DT-BK-HD-ZMD
```

Shared family:

```text
FM270X19DT
```

Variation-specific chunks follow.

---

## Family Base Rules

A family identifier should:

1. be shared by all variations
2. uniquely identify a product family
3. not appear across unrelated products

This creates deterministic grouping.

---

# Duplicate SKU Discovery

Originally duplicates were treated as errors.

Later discovery:

```text
Same SKU
    ↓
Amazon listing
eBay listing
Website listing
```

can legitimately create duplicate rows.

Therefore:

```text
duplicate sku
```

is not automatically an issue.

---

## Actionable Duplicate Cases

Only flag duplicates when:

* prices differ
* descriptions differ
* dimensions differ
* weights differ

Price differences are the most important.

---

# Pricing Learnings

## WooCommerce Pricing

Historically used for storefront pricing.

---

## Veeqo Pricing

Originally considered unreliable because:

* duplicated rows
* channel-specific prices
* sync artifacts

---

## Current Position

Veeqo may actually be closer to current operational pricing.

Especially where:

```text
WooCommerce
```

has become stale.

This remains an open question to discuss with the owner.

---

# Description Learnings

Many WooCommerce descriptions contain only:

```text
Lens Width
Bridge Width
Temple Length
```

specifications.

Veeqo often contains:

* marketing copy
* customer-facing descriptions
* richer product information

Examples:

```text
proSPORT Liverpool Smoke Bifocal
proSPORT Garnet Gradient Bifocal
proSPORT Orchid Lady Polarized Bifocal
```

Veeqo descriptions were significantly better.

Potential enrichment source.

---

# Category Work

Owner manually categorized products.

Large category taxonomy now exists.

Key rule:

```text
reuse existing categories
```

not:

```text
invent new categories
```

Claude handoff instructions created to:

* fill missing categories only
* preserve existing decisions
* mark suggestions for review

---

# Veeqo Cleanup Plan

Before building storefronts:

## 1

Investigate duplicate SKU rows.

Determine:

* same listing duplicated?
* multiple channel copies?
* genuine conflicts?

---

## 2

Enforce family SKU structure.

Goal:

```text
all variations in a product
share same family base
```

---

## 3

Review price conflicts.

Surface:

* all observed prices
* highest price
* lowest price

Owner decides canonical value.

---

## 4

Identify products absent from Veeqo.

Review manually.

Determine:

* discontinued?
* website-only?
* never synced?

---

# Future Architecture

Desired flow:

```text
Amazon
eBay
      ↓
    Veeqo
      ↓
   Supabase
      ↓
Next.js Storefronts
```

Storefronts should not become inventory managers.

Inventory should continue being managed through:

```text
Amazon
eBay
Veeqo
```

The custom websites should consume inventory data, not become the source of truth.

This avoids rebuilding inventory synchronization logic already solved by Veeqo.

---

# Biggest Lesson

The migration was never really about moving WooCommerce products.

It became a process of:

```text
finding trustworthy inventory data
```

and building systems that can distinguish:

```text
real sellable products
```

from

```text
years of accumulated ecommerce noise.
```

