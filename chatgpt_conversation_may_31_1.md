# BikerShades Frontend Build Log (May 31, 2026)

*Last updated: May 31, 2026*

## Overview

This document summarizes the frontend architecture, API contract decisions, implementation progress, bug fixes, and future work completed during the BikerShades storefront rebuild.

---

# Architecture Decisions

## Frontend Stack

* Next.js (App Router)
* TypeScript
* Tailwind CSS

Not included initially:

* ESLint
* Prettier
* Cart system
* Checkout
* User accounts

The goal was to build the storefront foundation first and add tooling later.

---

# Backend Contract Review

Several API contract improvements were made before frontend development began.

## Removed Category Detail Endpoint

Removed:

```txt
POST /api/public/categories/detail
```

Reason:

* Category tree already contains all required information.
* Breadcrumbs can be generated client-side.
* Slug lookups violated the "IDs are source of truth" rule.

Final category endpoints:

```txt
POST /api/public/categories/tree
```

Only category endpoint exposed publicly.

---

## Pagination Improvements

Added to product search response:

```ts
page
limit
totalPages
totalProducts
hasNextPage
hasPreviousPage
```

Backend now returns the actual clamped limit used.

Clamping rules:

```txt
page >= 1
limit = 1–100
```

---

## CamelCase Public API

Public responses now use camelCase consistently.

Examples:

```ts
minPriceCents
maxPriceCents
salePriceCents
productId
variationId
```

Database remains snake_case internally.

---

## Product Card Sale Pricing

Added:

```ts
salePriceCents
```

to ProductSummary.

Card rendering rules:

### Product-level sale

```txt
regular price
sale price
```

### Variation-level sale

```txt
Sale badge
regular range
```

No additional detail requests required.

---

## Variation Attribute Contract

Original contract:

```ts
["Color: Black Mirror Blue"]
```

Improved to:

```ts
[
  {
    name: "Color",
    value: "Black Mirror Blue"
  }
]
```

Frontend no longer parses strings.

---

## Product Attributes Cleanup

Discovered:

```ts
product.attributes
```

contained option metadata rather than specifications.

Example:

```ts
[
  {
    name: "color",
    terms: [...]
  }
]
```

This field was:

* Used internally by backend
* Removed from public API
* Removed from handoff

Future specifications should use a dedicated field.

---

# Storefront Foundation

## Brand Abstraction

Created:

```txt
src/lib/brand.ts
```

```ts
export const BRAND = {
  slug: "bikershades",
  name: "BikerShades",
  logo: ""
};
```

Removed all hardcoded:

```txt
bikershades
BikerShades
```

references.

---

## Shared Libraries

### src/lib/constants.ts

Contains:

```ts
API_BASE
DEFAULT_PAGE_LIMIT
```

### src/lib/types.ts

Contains all API types.

### src/lib/api.ts

Contains:

```ts
fetchCategoryTree()
searchProducts()
fetchProductDetail()
```

### src/lib/categories.ts

Contains:

```ts
buildCategoryMap()
getBreadcrumbs()
```

---

# Category Architecture

## Category Context

Originally:

```txt
Client-side fetch
useEffect
Loading state
Hydration rerender
```

Refactored to:

```txt
Server-side fetch in layout.tsx
CategoryProvider(initialTree)
Immediate availability
```

Benefits:

* No hydration flash
* No client request
* Better performance
* Cached for 1 hour

---

# Navigation System

Implemented recursive category navigation.

Rules:

## Root Category With Children

```txt
Button
Opens dropdown
No immediate navigation
```

Examples:

* Sunglasses
* Bifocals
* Prescription
* Transitions

---

## Child Category With Children

```txt
Link
Can navigate
Shows flyout menu
```

Examples:

```txt
Sunglasses
 └ Brands
     └ Wiley X
```

---

## Leaf Categories

Direct navigation.

Examples:

```txt
Sale
Combo Sets
Try Before You Buy
```

---

# Listing Pages

Implemented:

```txt
/products
/category/[categoryId]
/search
```

Features:

* Pagination
* Sorting
* Sale filter
* Stock filter
* Search integration
* Empty states
* Error handling

---

# Product Cards

Built:

```txt
ProductCard
```

Supports:

* Thumbnail
* Name
* Sale badge
* Price display
* Stock state

All pricing logic follows the backend contract.

---

# Product Detail Page

Implemented:

```txt
/ product / [productId]
```

Features:

## Variation Selection

Supports:

```txt
Color
Lens
Transition Type
```

Selection updates:

* Variation
* SKU
* Price
* Stock
* Images

---

## Image Gallery

Uses:

```txt
variation.images
```

if available.

Falls back to:

```txt
productImages
```

otherwise.

---

## Breadcrumbs

Generated from:

```ts
CategoryMap
```

using category IDs.

Never uses slugs.

---

## Description

Supports:

* HTML content
* Bullet summaries
* Description images

---

# Major Bug Fixes

## Variation Selection Bug

Problem:

Variation attributes were returned as:

```ts
{
  name: "black-mirror-green",
  value: "black-mirror-green"
}
```

instead of:

```ts
{
  name: "Color",
  value: "Black Mirror Green"
}
```

Result:

* Selector grouped incorrectly
* Selected variation never changed

Fix:

Backend now maps variation attribute slugs using product attribute definitions.

---

## Specifications Bug

Problem:

Specifications rendered:

```txt
0 | color
1 | transition-type
```

Cause:

Frontend treated an attribute-definition array as a specs object.

Fix:

Hide Specifications section when:

```ts
Array.isArray(product.attributes)
```

Later:

* product.attributes removed from public API entirely.

---

## Category Context Fetch Bug

Problem:

Client-side fetch attempted:

```txt
/api/public/categories/tree
```

from browser.

Result:

```txt
Failed to fetch
```

Fix:

Move category tree loading into:

```txt
app/layout.tsx
```

server-side.

---

# Homepage

Implemented:

```txt
/
```

Sections:

## Hero

* Brand title
* Search bar
* Browse products CTA

## Featured Categories

* Sunglasses
* Bifocals
* Prescription
* Transitions

## Featured Products

First 8 products.

## Category Discovery

Shows category tree hierarchy.

---

# Current Status

## Complete

* API contract finalized
* Category tree
* Search
* Product listings
* Product detail page
* Variation selection
* Breadcrumbs
* Brand abstraction
* Homepage foundation

---

# Template Milestone

🚨 STORE FRONT TEMPLATE CREATED HERE 🚨

This is the recommended point to create the reusable storefront base.

Why:

* Brand values isolated
* Navigation complete
* Product flow complete
* API contract stable

Before adding:

* Custom branding
* Hero graphics
* Marketing content
* Brand-specific layouts

---

# Next Steps

## Immediate

* Fix homepage category counts
* Improve dropdown contrast
* Polish homepage layout

## Near Future

* Create storefront template repository
* Duplicate for additional brands
* Add brand-specific styling

## Later

* Cart
* Checkout
* Customer accounts
* Wishlist
* Reviews
* SEO enhancements
* Analytics

---

# Final Assessment

The storefront foundation is complete.

The application now functions as a reusable multi-brand catalog platform with:

* Shared API layer
* Shared category architecture
* Shared product detail system
* Shared homepage structure
* Brand-specific configuration layer

Future brands should be created by copying the storefront template and changing:

```ts
BRAND.slug
BRAND.name
BRAND.logo
```

without modifying the underlying storefront architecture.

