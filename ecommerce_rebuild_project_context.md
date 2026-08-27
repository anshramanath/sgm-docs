# Ecommerce Rebuild Project Context & Findings (June 4th, 2026)

## Project Overview

The goal is to rebuild three legacy ecommerce storefronts:

* ProSportSunglasses
* Sunglass Monster
* BikerShades

The original plan was to migrate the existing WooCommerce catalogs into a modern stack. During the migration effort it became clear that the catalogs contain years of accumulated technical debt, duplicate products, administrative products, discontinued products, and inconsistencies between systems.

To avoid importing years of bad data into the rebuilt platform, a full validation and migration pipeline was built.

---

# Migration Pipeline

The pipeline currently operates as:

```text
fetch
→ validate_products
→ validate_variations
→ reshape
→ create_items
```

The purpose of the pipeline is not merely to transform data.

The purpose is to determine:

* Which products are real products
* Which products are actively sold
* Which products contain data quality problems
* Which products should not be migrated

Products that fail validation are flagged for review rather than automatically imported.

---

# Product Funnel

## WooCommerce → create_items

| Stage                | ProSport | Monster | BikerShades |
| -------------------- | -------: | ------: | ----------: |
| Fetched Products     |      171 |     277 |         704 |
| Validated Products   |      169 |     267 |         611 |
| Fetched Variations   |    1,758 |   2,675 |       5,993 |
| Validated Variations |    1,669 |   2,560 |       5,299 |
| create_items Passed  |      161 |     260 |         461 |

Total Product Loss:

| Brand       | Products Lost | Percent Lost |
| ----------- | ------------: | -----------: |
| ProSport    |            10 |           6% |
| Monster     |            17 |           6% |
| BikerShades |           243 |          35% |

BikerShades has a much larger amount of catalog debt than the other two stores.

---

# Why Products Were Removed

## ProSport

Products failed primarily because:

* Draft products
* Missing SKUs
* Mixed product families grouped together

Examples:

* Sharkbait Polarized Float
* Chipper Aviator
* Eliminator Polarized Bifocal
* First Lady Bifocal
* Kumatage Bifocal
* Cooper Bifocal
* Morocco Mirror

These products contained variations whose SKU structure indicated multiple unrelated products had been grouped into a single WooCommerce listing.

---

## Sunglass Monster

Products failed because:

* Missing SKUs
* Missing images
* Private products
* Attribute collisions
* Mixed product families

Examples:

* Sharkbait Polarized Float
* Chipper Aviator
* Harvard Multifocal Reader
* First Lady Bifocal
* Kumatage Bifocal
* Morocco Mirror

These products also exhibited SKU-family inconsistencies.

---

## BikerShades

BikerShades contained substantially more non-product records.

Examples include:

* RX services
* Try Before You Buy deposits
* Warranty products
* USPS return labels
* Administrative listings
* Phone-order products
* Legacy accessory listings

Many of the excluded products were not customer-facing products at all.

---

# Major Catalog Discovery

## Mixed Product Families

The single largest source of data loss across all three stores was WooCommerce products that combine multiple actual product families.

Examples:

### Polarized vs Non-Polarized

```text
SP73X31EL
SPPL73X31EL
```

### Old SKU vs New SKU

```text
BF4X33FL
BF4X33FLCT
```

### Multiple Frame Models

Several BikerArmour products contain multiple distinct frame models grouped under one WooCommerce product.

These products were intentionally flagged rather than automatically split because the correct grouping cannot be determined safely from the data alone.

---

# SKU Learnings

Variation SKUs are generally trustworthy.

WooCommerce parent SKUs are often not.

Examples of parent SKU patterns:

```text
-0
-0-XT
-0-SM
-COMBOS
-Master-MFB
```

However:

```text
7EYE-ASPEN-0-HIGHRX
FM270X20TX-0-EC
```

are legitimate simple-product SKUs.

Therefore:

* Variable product parent SKUs cannot be blindly trusted.
* Simple-product SKUs must be preserved exactly.

---

# Combo Product Discovery

A major finding involved combo products.

Example:

```text
PR29X36ST
CPR29X36ST
```

The leading:

```text
C
```

indicates a combo version of the same product.

These products were originally being incorrectly flagged.

The validation pipeline was updated to treat:

```text
CPR...
PR...
```

as belonging to the same product family.

This restored:

* 12 BikerShades products
* 437 BikerShades variations

that would otherwise have been excluded.

---

# Veeqo Investigation

A major goal was determining whether Veeqo could become the primary source of truth.

Initial assumption:

```text
WooCommerce = legacy
Veeqo = source of truth
```

The investigation produced mixed results.

---

## What Veeqo Does Well

Veeqo is extremely useful for SKU existence validation.

Coverage:

| Brand            | Veeqo Match Rate |
| ---------------- | ---------------: |
| BikerShades      |             ~99% |
| Sunglass Monster |             ~98% |
| ProSport         |             ~10% |

This indicates that BikerShades and Monster are heavily represented in Veeqo while ProSport likely follows a different inventory workflow.

---

## What Veeqo Does Poorly

### Inventory

The exported inventory data is mostly empty.

### Product Relationships

Veeqo operates primarily at the SKU/sellable level and does not clearly preserve storefront product groupings.

### Duplicate Records

Because WooCommerce is connected as a channel, many products appear multiple times.

Results:

* 431 duplicate SKUs
* 162 duplicate SKUs with conflicting prices

This makes it difficult to determine which Veeqo record is authoritative.

---

# Veeqo Description Findings

One of the most surprising discoveries was the difference in descriptions.

Example:

## WooCommerce

```text
Lens Width
Bridge Width
Temple Length
Lens Height
Frame Width
```

## Veeqo

```text
Customer-facing marketing copy
Benefits
Use cases
Product positioning
```

Approximately 241 products contain significantly better descriptions in Veeqo than in WooCommerce.

This creates a possible hybrid strategy:

```text
WooCommerce → specifications
Veeqo → marketing description
```

---

# Price Investigation

This remains unresolved.

Findings:

* Approximately 27% of matched SKUs have price differences.
* Median difference is roughly $3.
* Maximum difference is roughly $28.

Possible explanations:

1. WooCommerce prices are stale.
2. Veeqo prices are stale.
3. Marketplace pricing differs intentionally from website pricing.
4. Different channels maintain independent pricing.

Further clarification from the operations team is required.

---

# Bifocal & Reader Findings

Most Veeqo misses fall into a very specific category.

Out of 15 Veeqo-flagged products:

* 13 are bifocals or readers.
* 2 are regular sunglasses.

Patterns:

### Entire Product Missing

Examples:

* Verbena
* Bypass
* Gullwing
* Brooklyn

These products have no matching Veeqo inventory at all.

### High-Power Variations Missing

Examples:

* Sharkbait
* Cooper
* Hawkeye HD
* Cabot
* Edison

Lower powers exist in Veeqo.

Higher powers (2.25–3.00+) do not.

Possible explanation:

These niche strengths were never added to inventory management.

---

# Amazon Findings

Amazon hosts its own images.

Product images are served from:

```text
m.media-amazon.com
```

This means Amazon is not dependent on WooCommerce image hosting.

Removing WooCommerce does not automatically break Amazon listings.

---

# BikerShades Programs That Should Become Features

## Try Before You Buy

Current implementation:

* Dozens of WooCommerce products

Recommended implementation:

* One configurable Try Before You Buy workflow
* Customer selects frames
* Customer pays deposit

---

## RX Frame Submission

Current implementation:

* Individual RX products per frame

Recommended implementation:

* One RX submission flow
* Customer selects frame
* Customer uploads prescription

This eliminates over 100 catalog records while preserving functionality.

---

# Current Understanding

The project originally began as:

```text
Migrate WooCommerce
```

It has evolved into:

```text
Determine the authoritative source for each piece of product data.
```

Current understanding:

| Data Type               | Current Best Source  |
| ----------------------- | -------------------- |
| Product Grouping        | WooCommerce          |
| Variation Relationships | WooCommerce          |
| Categories              | WooCommerce          |
| Images                  | WooCommerce / Amazon |
| Specifications          | WooCommerce          |
| SKU Validation          | Veeqo                |
| Marketing Descriptions  | Possibly Veeqo       |
| Pricing                 | Unresolved           |
| Inventory               | Unresolved           |

---

# Key Questions For The Team

1. When a new product is added, where is it created first?
2. When a price changes, where is it updated?
3. Which system should be trusted for inventory?
4. Are Veeqo descriptions considered the current descriptions?
5. Why is ProSport largely absent from Veeqo?
6. Which products should intentionally not be migrated?
7. Are bifocals/readers missing from Veeqo still actively sold?
8. How are products assigned to ProSport, Sunglass Monster, and BikerShades?

Answering these questions will determine the final migration architecture and the correct source of truth for each field.
