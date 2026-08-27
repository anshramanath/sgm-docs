# Veeqo Strategy and Next Steps (June 5, 2026)

*Last Updated: 2026-06-05*

---

# Where We Started

The original assumption was:

```text
WooCommerce
    ↓
Source of Truth
    ↓
New Website
```

The migration project was focused on extracting products from:

* ProSportSunglasses
* Sunglass Monster
* BikerShades

and rebuilding the storefronts using cleaned WooCommerce data.

---

# What We Learned

After building the migration pipeline and validating thousands of products and variations, a different picture emerged.

WooCommerce is useful for:

* Product discovery
* Variation discovery
* Product grouping
* Category structure
* Images
* Product specifications

But WooCommerce is not necessarily the best long-term operational system.

---

# Veeqo Discovery

Veeqo sits in the middle of:

```text
Amazon
eBay
Other Channels
    ↓
  Veeqo
```

and already handles:

* Inventory synchronization
* Marketplace listings
* Orders
* Channel management

This means rebuilding the websites does NOT require rebuilding inventory management.

---

# Key Realization

The websites should not become the source of truth.

Bad architecture:

```text
Amazon
eBay
    ↓

Website
    ↓

Veeqo
```

because every inventory update, catalog update, and marketplace update would need to originate from the website.

That turns the website into an ERP system.

---

# Preferred Architecture

```text
Amazon
eBay
    ↓

Veeqo
    ↓

Websites
```

In this model:

Veeqo owns:

* Inventory
* Marketplace synchronization
* Product operations
* Orders

The websites own:

* Presentation
* Search
* Navigation
* Customer experience

The websites become consumers of catalog data instead of maintaining the catalog themselves.

---

# Problem #1: Duplicate SKUs

Current Veeqo findings:

```text
431 duplicate SKUs
162 duplicate SKUs with conflicting prices
```

Likely cause:

```text
Amazon
    ↓
Veeqo Product

WooCommerce
    ↓
Veeqo Channel Sync

Same SKU
    ↓
Multiple Rows
```

Questions to answer:

* Why do duplicate SKUs exist?
* Which record is authoritative?
* Are these true duplicates or channel-specific copies?
* Can duplicates be merged safely?

Before any migration work continues, the Veeqo data model should be understood.

---

# Problem #2: Product Grouping

WooCommerce contains:

```text
Product
    ↓
Variations
```

relationships.

Veeqo appears to expose products primarily as sellable SKUs.

Example:

```text
FM270X20TX-BK-ECCL-ZMD
FM270X20TX-BK-CL-ZMD
FM270X20TX-RD-ECCL-ZMD
```

These clearly belong together, but Veeqo does not appear to formally represent that relationship.

---

# Proposed Solution

Introduce the concept of a:

```text
family_sku
```

Every variation SKU should be:

```text
family_sku
    +
variation chunks
```

Example:

```text
FM270X20TX-BK-ECCL-ZMD
FM270X20TX-BK-CL-ZMD
FM270X20TX-RD-ECCL-ZMD
```

Family SKU:

```text
FM270X20TX
```

Variation-specific chunks:

```text
BK
ECCL
ZMD
```

---

# Family SKU Requirements

A family SKU should satisfy two rules:

## Rule 1

Every variation belonging to a product shares the same family SKU.

Example:

```text
FM270X20TX-BK-ECCL-ZMD
FM270X20TX-BK-CL-ZMD
FM270X20TX-RD-ECCL-ZMD
```

All belong to:

```text
FM270X20TX
```

---

## Rule 2

A family SKU must be globally unique.

Example:

```text
FM270X20TX
```

should identify exactly one product family.

Not:

```text
Product A
Product B
```

simultaneously.

---

# Why This Matters

If family SKUs are enforced:

```text
family_sku
    ↓
many variation SKUs
```

then:

* Product grouping becomes deterministic
* Variation grouping becomes deterministic
* Duplicate detection becomes easier
* Catalog maintenance becomes easier
* Storefront generation becomes easier

The catalog becomes self-describing.

---

# What the Migration Pipeline Already Found

The migration pipeline effectively discovered many violations of this rule.

Examples:

### Mixed Product Families

```text
CA10X15...
CA10X16...
```

inside the same WooCommerce product.

### Polarized vs Non-Polarized

```text
SP73X31EL...
SPPL73X31EL...
```

inside the same WooCommerce product.

### SKU Revisions

```text
BF4X33FL
BF4X33FLCT
```

inside the same WooCommerce product.

These products were flagged because they violate the expectation that all variations in a product should share a common family base.

---

# Product Data Strategy

Current thinking:

## Product Grouping

Derived from:

```text
family_sku
```

---

## Variations

Derived from:

```text
variation chunks
```

following the family SKU.

---

## Descriptions

Potentially sourced from Veeqo.

Many products have significantly better customer-facing descriptions in Veeqo than in WooCommerce.

---

## Specifications

Likely sourced from WooCommerce.

WooCommerce often contains:

* Lens width
* Bridge width
* Temple length
* Frame width

which should remain available.

---

## Images

Amazon-hosted images appear reliable.

Potential sources:

* Amazon
* Veeqo
* WooCommerce

depending on completeness.

---

## Prices

Still unresolved.

Need confirmation regarding:

* Amazon pricing
* eBay pricing
* WooCommerce pricing
* Veeqo pricing

and which should drive the rebuilt storefronts.

---

## Inventory

Long-term goal:

```text
Veeqo
    ↓
Inventory Source
```

rather than maintaining inventory separately in the websites.

---

# Physical Product Data

All sunglasses appear to ship in roughly the same form factor.

Weight and dimensions can likely be standardized and attached automatically rather than relying on inconsistent legacy data.

This should be confirmed with the operations team.

---

# Missing Products

Current Veeqo misses are relatively small:

```text
ProSport: 7
Monster: 5
BikerShades: 3
```

Most are:

* Bifocals
* Readers
* High-power reader variants

Questions:

* Are these discontinued?
* Were they never added to Veeqo?
* Are they website-only products?

---

# Long-Term Architecture

Target architecture:

```text
Amazon
eBay
Other Channels
      ↓

    Veeqo
      ↓

Canonical Catalog
      ↓

Supabase
      ↓

Websites
```

The websites should consume catalog data.

They should not become the operational source of truth.

---

# Immediate Next Steps

1. Meet with operations team.
2. Understand Veeqo duplicate SKU behavior.
3. Determine whether duplicate channel records can be merged.
4. Confirm family SKU conventions.
5. Determine authoritative price source.
6. Determine authoritative description source.
7. Resolve products missing from Veeqo.
8. Standardize physical product dimensions.
9. Finalize catalog structure.
10. Import catalog into database.
11. Build storefronts on top of the catalog.

---

# Core Principle

The goal is not:

```text
Build another inventory system.
```

The goal is:

```text
Use Veeqo as the inventory and operations platform,
and build modern storefronts that consume that data.
```

This minimizes maintenance, reduces operational overhead, and keeps Amazon, eBay, Veeqo, and the websites synchronized through a single catalog workflow.

