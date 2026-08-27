# Bikershades Rebuild Journal (May 27, 2026)

This document captures everything learned, discovered, designed, and planned while investigating and rebuilding the Bikershades website.

## Core Journey

Started by inspecting the storefront and discovering it was powered by WordPress + WooCommerce.

Key discoveries:
- Product pages exposed WooCommerce classes.
- Images were stored under `/wp-content/uploads/...`.
- WordPress exposed a public API at `/wp-json/`.
- WooCommerce exposed product data at `/wp-json/wc/store/products`.

This shifted the project from a scraping problem into a migration problem.

## WooCommerce API Discovery

Discovered:

`/wp-json/`

which revealed WooCommerce namespaces:

- wc/store
- wc/store/v1

Then discovered:

`/wp-json/wc/store/products`

which returns an array of product objects containing:
- id
- name
- slug
- descriptions
- images
- prices
- categories
- stock information
- attributes
- variations

The storefront is rendering these products from the same underlying data model.

## Pagination

Learned that the endpoint is paginated.

Plan:

Fetch pages until:

```python
if products == []:
    break
```

Store everything in:

`products.json`

## JSON Learnings

JSON is:
- data only
- objects
- arrays
- strings
- numbers
- booleans
- null

No functions, variables, loops, or comments.

JSON.parse converts:

Text
↓
Native objects

Example:

'[{"name":"Backspin"}]'

becomes:

[{ name: "Backspin" }]

## Migration Architecture

Final migration pipeline:

WooCommerce API
↓
products.json
↓
download images
↓
clean_products.json
↓
database
↓
new storefront

## Why Save products.json

Acts as:
- backup
- source of truth
- debugging artifact
- rerunnable import source

## Image Migration

Do not rely on:

https://bikershades.com/wp-content/uploads/...

Instead:

Old image
↓
Download
↓
Upload to own storage
↓
Store new URL

Reasons:
- independence
- no hotlinking risk
- no broken images if old site changes

## Data Reshaping

WooCommerce schema ≠ application schema.

Example:

WooCommerce:

{
  "prices": {
    "price": "1999"
  }
}

Desired:

{
  "priceInCents": 1999
}

Transform:
API model
↓
Application model

## Supabase + RLS Learnings

Major concepts learned:

### JWT

Frontend obtains:

session.access_token

and sends:

Authorization: Bearer TOKEN

to backend routes.

### getUser()

Purpose:

Verify JWT and return associated user.

Does NOT:
- log in the client
- attach token to future queries
- modify auth state

### RLS

RLS lives inside Postgres.

Without matching policies:

- SELECT returns no rows
- UPDATE affects 0 rows
- INSERT denied
- DELETE denied

Rows effectively appear not to exist.

### USING vs WITH CHECK

USING:
Can I touch this row?

WITH CHECK:
Can the resulting row exist?

Ownership example:

USING:
User owns row → pass

WITH CHECK:
User changes owner_id → fail

### Service Role vs Anon

Service role:
- bypasses RLS

Anon + JWT:
- follows RLS

Key insight:

Postgres evaluates:

Who is making the request?

not:

What query was written?

### Backend Queries

If JWT is attached to the anon client:

select()
↓
RLS automatically filters rows

Meaning many SELECT and UPDATE operations do not require getUser().

### Update Example

Update query:

update(...)
↓
RLS
↓
only user-owned rows updated

No manual owner filter required.

### Service Role Philosophy

Use anon + JWT for:
- user CRUD
- ownership-protected operations

Use service role for:
- migrations
- webhooks
- admin tools
- bulk imports
- scheduled jobs

## Amazon Investigation

Discovered Bikershades also sells on Amazon.

Potential future integration:

Amazon Selling Partner API (SP-API)

Potential MCP tools:

- list_products
- update_inventory
- update_price
- create_listing
- delete_listing

Learned that Seller Central does not automatically mean API access is already configured, but an active seller can generally authorize an SP-API application.

## MCP Vision

Possible future architecture:

Claude
↓
MCP Server
↓
Amazon SP-API

Allowing AI-powered inventory management.

## Planned Rebuild

Phase 1:
Fetch products

Phase 2:
Download images

Phase 3:
Normalize products

Phase 4:
Design Supabase schema

Phase 5:
Upload images

Phase 6:
Import records

Phase 7:
Build storefront

Phase 8:
Potential Amazon integration

## Biggest Takeaways

1. WooCommerce exposed nearly everything needed through APIs.
2. This became a migration problem rather than a scraping problem.
3. Data migration is largely Extract → Transform → Load.
4. RLS is authorization inside the database.
5. Service role should be used intentionally, not avoided entirely.
6. Ownership rules can often be delegated to Postgres.
7. Modern web apps are mostly APIs + databases underneath.
8. Rebuilding Bikershades became an exercise in ecommerce architecture, migration pipelines, authentication, authorization, and future MCP-based commerce tooling.

note: produced bikershades_migration_guide_may_27.md
