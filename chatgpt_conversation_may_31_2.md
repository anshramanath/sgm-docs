# Sunglass Monster / BikerShades Backend Build Log (May 31, 2026)

## Overview

This project is a multi-brand eyewear platform called **Sunglass Monster**.

The first brand being launched is:

```txt
BikerShades
```

The architecture is designed so that multiple brands can share the same backend while having independent storefronts.

Examples:

```txt
BikerShades
Sunglass Monster
Brand X
Brand Y
```

Each storefront is its own frontend while all brands share a common database and backend.

---

# Tech Stack

## Frontend

```txt
Next.js App Router
TypeScript
```

## Backend

```txt
Next.js Route Handlers
Supabase Postgres
Supabase Storage
Supabase Auth
```

---

# Security Architecture

Originally RLS was disabled.

After discussion, the architecture was changed.

## Final Decision

Enable RLS on all tables.

The browser never talks directly to tables.

Everything goes through server endpoints.

---

## Browser Client

Used only for authentication.

```ts
createBrowserClient(...)
```

Responsibilities:

* sign in
* sign out
* getSession()
* getUser()

The browser never performs table queries.

---

## Server Client

Used inside Server Components.

```ts
createServerClient(...)
```

Responsibilities:

* read auth cookies
* determine current user
* protect pages

Example:

```ts
const {
  data: { user },
} = await supabase.auth.getUser();

if (!user) redirect("/login");
```

---

## Admin Client

Used inside API routes.

```ts
createClient(
  SUPABASE_URL,
  SUPABASE_SERVICE_ROLE_KEY
)
```

Responsibilities:

* bypass RLS
* access all tables
* perform reads and writes

All database access happens through this client.

---

# Authentication Lessons Learned

## getSession()

Returns locally stored session data.

No network request.

Can be stale.

---

## getUser()

Validates token with Supabase.

Makes a network request.

Cannot be stale.

Use this whenever authorization matters.

---

## Protected Pages

Decision:

```txt
Server Components
```

instead of:

```txt
Middleware
```

Reason:

* easier to reason about
* explicit protection
* avoids accidentally protecting routes

Example:

```ts
const supabase = await createServerClient();

const {
  data: { user },
} = await supabase.auth.getUser();

if (!user) redirect("/login");
```

---

# Admin Authorization

Table:

```sql
create table admins (
  user_id uuid primary key references auth.users(id)
);
```

Admin check:

```ts
const { data: admin } = await supabaseAdmin
  .from("admins")
  .select("user_id")
  .eq("user_id", user.id)
  .single();

if (!admin) redirect("/unauthorized");
```

---

# Database Schema

## brands

```sql
id
name
slug
```

Example:

```txt
BikerShades
```

---

## categories

```sql
id
brand_id
parent_id
name
slug
```

Important rules:

* category IDs are source of truth
* category names can duplicate
* category slugs can duplicate
* products only belong to leaf categories

Frontend displays:

```txt
category.name
```

Backend uses:

```txt
category.id
```

---

## products

```sql
id
brand_id
name
slug
sku
description
summary
attributes
sale
min_price_cents
max_price_cents
sale_price_cents
stock
```

Important:

```txt
product.id = source of truth
```

Product slugs are NOT unique.

Product lookups never use slug.

---

## product_categories

Many-to-many table:

```sql
product_id
category_id
```

---

## variations

```sql
id
product_id
sku
attribute
stock
```

Variation IDs are source of truth.

Variations are returned sorted by:

```sql
sku ASC
```

---

## product_images

```sql
id
product_id
src
sort_order
```

---

## variation_images

```sql
id
variation_id
src
sort_order
```

---

## description_images

```sql
id
product_id
src
sort_order
```

---

# Public API Philosophy

Frontend never queries tables directly.

Frontend only calls public endpoints.

All endpoints use:

```txt
POST
```

Request bodies only.

No query parameters.

Response format:

```json
{
  "success": true,
  "data": ...
}
```

Errors:

```json
{
  "success": false,
  "error": "message"
}
```

---

# Public Endpoints

## Brands

```txt
POST /api/public/brands
```

Returns:

```json
[
  {
    "id": "...",
    "name": "BikerShades",
    "slug": "bikershades"
  }
]
```

Tested successfully.

---

## Category Tree

```txt
POST /api/public/categories/tree
```

Request:

```json
{
  "brandSlug": "bikershades"
}
```

Returns nested category hierarchy.

Used for:

* navbar
* mega menu
* sidebar filters

Parent categories contain children.

Category IDs are returned.

---

## Category Detail

```txt
POST /api/public/categories/detail
```

Returns:

```json
{
  "id": "...",
  "parent": {},
  "children": [],
  "productCount": 0
}
```

Used for:

* breadcrumbs
* category pages

---

## Product Search

```txt
POST /api/public/products/search
```

Request:

```json
{
  "brandSlug": "bikershades",
  "categoryId": "uuid",
  "search": "wiley",
  "page": 1,
  "limit": 24,
  "saleOnly": false,
  "inStockOnly": false,
  "sort": "name_asc"
}
```

Features:

* pagination
* sorting
* category filtering
* descendant expansion
* stock filtering
* sale filtering
* search
* thumbnails

Searches:

```txt
name
sku
```

Category filtering uses:

```txt
categoryId
```

not slug.

---

## Product Detail

```txt
POST /api/public/products/detail
```

Request:

```json
{
  "brandSlug": "bikershades",
  "productId": "uuid"
}
```

Important:

```txt
productId
```

replaced:

```txt
productSlug
```

because product slugs are not unique.

Returns:

```json
{
  "product": {},
  "variations": [],
  "categories": [],
  "productImages": [],
  "descriptionImages": []
}
```

---

# Product Detail Improvements

Variation images are nested directly into variations.

Example:

```json
{
  "id": "...",
  "sku": "...",
  "images": [
    {
      "id": "...",
      "src": "..."
    }
  ]
}
```

No frontend joining required.

---

# API Testing Results

## Brands Endpoint

Passed.

Returned:

```txt
BikerShades
```

correctly.

---

## Product Search Endpoint

Passed.

Returned:

```txt
573 products
115 pages
```

Pagination works.

Thumbnail lookup works.

Filtering works.

---

## Product Detail Endpoint

Passed.

Verified:

* product lookup
* category loading
* variation loading
* product images
* description images
* variation images

A product with variation images was tested and returned nested image data correctly.

---

# Final Architecture

Frontend only knows about:

```txt
POST /api/public/brands
POST /api/public/categories/tree
POST /api/public/categories/detail
POST /api/public/products/search
POST /api/public/products/detail
```

Frontend never accesses tables directly.

Backend owns all database logic.

RLS remains enabled.

Service role is used exclusively inside server endpoints.

---

# Current Status

Backend public API is complete enough to begin building:

```txt
Homepage
Navbar
Mega Menu
Category Pages
Search
Product Detail Pages
Product Galleries
Variation Selection
Breadcrumbs
Pagination
```

Next major milestone:

```txt
Build BikerShades.com frontend using the completed public API.
```

