# Supabase / Postgres Query Speed, Pagination, and Indexes (June 21, 2026)

## Big Picture

Supabase uses Postgres underneath. When you write a Supabase query like:

```ts
supabase
  .from("products")
  .select("*")
  .eq("brand_id", brandId)
  .range(0, 23)
```

Supabase is not fetching every row into your app and then filtering/paginating there.

It turns into a SQL query conceptually like:

```sql
SELECT *
FROM products
WHERE brand_id = ?
LIMIT 24 OFFSET 0;
```

Postgres handles the filtering, ordering, and limiting inside the database.

The important question is:

> How much work does Postgres have to do to find the rows?

Pagination controls how much comes back.  
Indexes control how efficiently Postgres can find the matching rows.

---

# Pagination

## What Pagination Does

Pagination means returning only a subset of rows.

Example:

```ts
supabase
  .from("products")
  .select("id, name, slug, min_price_cents")
  .eq("brand_id", brandId)
  .order("name")
  .range(0, 23)
```

This means:

> Give me products 0 through 23.

In SQL terms:

```sql
SELECT id, name, slug, min_price_cents
FROM products
WHERE brand_id = ?
ORDER BY name
LIMIT 24 OFFSET 0;
```

## Why Pagination Helps

Pagination helps because it reduces:

1. Network response size
2. JSON serialization from Postgres/Supabase
3. JSON parsing in the frontend
4. Browser memory usage
5. React rendering work
6. Sometimes database work, if the database can stop early

Without pagination, if a brand has 10,000 products, Postgres may have to return all 10,000 products.

With pagination, it may only return 24.

## Important Distinction

Pagination does not always mean Postgres only looks at 24 rows.

If the query is easy and indexed, Postgres may find the first 24 quickly and stop.

But if the query requires a bad filter or unindexed sort, Postgres may still have to scan/sort many rows before it knows which 24 rows to return.

Example:

```sql
SELECT *
FROM products
WHERE brand_id = ?
ORDER BY name
LIMIT 24;
```

If there is an index on `(brand_id, name)`, this can be fast.

But:

```sql
SELECT *
FROM products
WHERE LOWER(name) LIKE '%aviator%'
ORDER BY price_cents
LIMIT 24;
```

may be slow if the search and price ordering are not indexed properly.

## Offset Pagination Can Get Slower

This:

```ts
.range(0, 23)
```

is usually cheap.

This:

```ts
.range(2400, 2423)
```

becomes:

```sql
LIMIT 24 OFFSET 2400;
```

Postgres still has to walk past the first 2400 matching rows before returning the next 24.

For very large datasets, cursor/keyset pagination is better.

Example:

```ts
supabase
  .from("products")
  .select("id, name, slug")
  .eq("brand_id", brandId)
  .gt("name", lastSeenName)
  .order("name")
  .limit(24)
```

That means:

> Give me the next 24 products after this last seen name.

---

# Indexes

## What an Index Is

An index is a separate data structure stored beside the table.

The table stores the actual rows.

The index stores sorted shortcuts that point back to the real rows.

Mental model:

```txt
products table
row 1: proSPORT, "Baseball Glasses"
row 2: BikerShades, "Aviator"
row 3: SunglassMonster, "Retro"
row 4: BikerShades, "Foam Padded"
row 5: BikerShades, "Clear Lens"
```

If you create:

```sql
CREATE INDEX products_brand_name_idx
ON products (brand_id, name);
```

Postgres builds something conceptually like:

```txt
Index sorted by (brand_id, name)

BikerShades, Aviator        -> row 2
BikerShades, Clear Lens     -> row 5
BikerShades, Foam Padded    -> row 4
proSPORT, Baseball Glasses  -> row 1
SunglassMonster, Retro      -> row 3
```

Each index entry stores:

```txt
indexed values + pointer to real table row
```

The real table does not have to be physically sorted.

The index is the sorted shortcut.

---

# B-Tree Structure

The most common Postgres index type is a B-tree.

A B-tree is not a simple binary tree. It is a wide, shallow tree.

Conceptually:

```txt
Root page
  ├── Branch page
  │     ├── Leaf page
  │     └── Leaf page
  └── Branch page
        ├── Leaf page
        └── Leaf page
```

The root page helps Postgres choose the correct direction.

The branch pages narrow the search.

The leaf pages contain the actual sorted index entries and row pointers.

For this query:

```sql
SELECT *
FROM products
WHERE brand_id = 'BikerShades'
ORDER BY name
LIMIT 24;
```

Postgres can:

```txt
1. Start at the root of the index
2. Jump toward the BikerShades section
3. Land near the first BikerShades entry
4. Read BikerShades entries in name order
5. Follow row pointers to the real table rows
6. Stop after 24 rows
```

That is why indexes are fast.

The speed comes from eliminating huge sections of the table, not from scanning everything.

---

# Single-Column Indexes

Yes, you can index just one column.

Example:

```sql
CREATE INDEX products_brand_id_idx
ON products (brand_id);
```

This helps:

```sql
SELECT *
FROM products
WHERE brand_id = 'bikershades';
```

Mental model:

```txt
brand_id        row pointer
bikershades     row 1
bikershades     row 7
bikershades     row 20
prosport        row 2
prosport        row 9
sunglassmonster row 3
```

This is not exactly the same as grouping the real table beforehand.

The actual table is not rearranged.

The index is a separate sorted lookup structure.

## One-Column Index Mental Model

```txt
One-column index = fast lookup/filter on that column
```

Example:

```sql
CREATE INDEX ON products (brand_id);
```

Good for:

```sql
WHERE brand_id = ...
```

---

# Multi-Column / Composite Indexes

A composite index indexes multiple columns together.

Example:

```sql
CREATE INDEX products_brand_name_idx
ON products (brand_id, name);
```

This means the index is sorted by:

```txt
brand_id first
then name inside each brand
```

Conceptually:

```txt
BikerShades
  Aviator
  Clear Lens
  Foam Padded

proSPORT
  Baseball Glasses

SunglassMonster
  Retro
```

This index is great for:

```sql
WHERE brand_id = ...
ORDER BY name
```

because Postgres can jump to the brand section and the rows are already sorted by name.

## Leftmost Prefix Rule

Composite indexes work best from left to right.

This index:

```sql
CREATE INDEX ON products (brand_id, name, price_cents);
```

can help queries that use:

```txt
brand_id
brand_id + name
brand_id + name + price_cents
```

It is less helpful for:

```txt
name only
price_cents only
```

because the index is not globally sorted by name or price. It is sorted by brand first.

## Example

This index:

```sql
CREATE INDEX products_brand_name_idx
ON products (brand_id, name);
```

helps:

```sql
WHERE brand_id = ...
```

and helps even more:

```sql
WHERE brand_id = ...
ORDER BY name
```

But it does not help as much for:

```sql
WHERE name = 'Aviator'
```

because `name` is the second column, not the first.

---

# Does `(brand_id, name)` Help With Brand-Only Queries?

Yes.

This index:

```sql
CREATE INDEX products_brand_name_idx
ON products (brand_id, name);
```

can still help:

```sql
SELECT *
FROM products
WHERE brand_id = 'bikershades'
LIMIT 24;
```

because `brand_id` is the first column in the index.

Postgres can:

```txt
1. Jump to the BikerShades section
2. Read matching rows
3. Stop after 24
```

However, without `ORDER BY`, SQL does not guarantee which 24 rows you get.

This:

```sql
WHERE brand_id = 'bikershades'
LIMIT 24;
```

means:

> Give me any 24 BikerShades products.

This:

```sql
WHERE brand_id = 'bikershades'
ORDER BY name
LIMIT 24;
```

means:

> Give me the first 24 BikerShades products by name.

That second query is where `(brand_id, name)` really shines.

---

# Why Column Order Matters

These two indexes are different:

```sql
CREATE INDEX ON products (brand_id, name);
```

```sql
CREATE INDEX ON products (name, brand_id);
```

For this query:

```sql
WHERE brand_id = ...
ORDER BY name
```

`(brand_id, name)` is better.

Why?

Because the data is sorted like:

```txt
brand first
then name within that brand
```

So Postgres can jump to one brand and read rows in name order.

But `(name, brand_id)` is sorted like:

```txt
name first
then brand within that name
```

That is not ideal when the query starts with brand.

Rule:

> Put the columns that match your common filters/order first.

Usually equality filters come first, then ordering/range columns.

Example:

```sql
WHERE brand_id = ...
ORDER BY name
LIMIT 24
```

Good index:

```sql
(brand_id, name)
```

---

# Normal Scan vs Indexed Search

Without an index, Postgres may do a table scan:

```txt
check row 1
check row 2
check row 3
check row 4
...
check row n
```

That is close to O(n).

With an index on `brand_id`, Postgres does not search row by row until it happens to find a match.

It uses the tree to jump near the first matching brand quickly.

Then it scans the matching section until the brand changes.

Mental model:

```txt
jump to first BikerShades entry
read BikerShades entries
stop when brand_id changes
```

Better phrasing:

> With an index, Postgres quickly jumps to the first matching `brand_id`, reads the matching section, then stops once the indexed values move past that `brand_id`.

---

# `product_id, stock` vs `stock, product_id`

This came up with this index:

```sql
CREATE INDEX variations_product_stock_idx
ON variations (product_id, stock);
```

Whether this is correct depends on the query shape.

## Good For Product-First Lookup

This index is good for:

```sql
SELECT *
FROM variations
WHERE product_id = 'abc'
AND stock > 0;
```

because the query starts with one product.

Postgres can:

```txt
jump to this product_id
then check stock inside that product
```

## Good For Stock-First Lookup

If your query is:

```sql
SELECT *
FROM variations
WHERE stock > 0;
```

then this index may be better:

```sql
CREATE INDEX variations_stock_product_idx
ON variations (stock, product_id);
```

because the query starts with stock.

## Best For In-Stock Product Filtering

For a product listing page, the query is often conceptually:

```sql
SELECT *
FROM products p
WHERE p.brand_id = ...
AND EXISTS (
  SELECT 1
  FROM variations v
  WHERE v.product_id = p.id
  AND v.stock > 0
);
```

This asks:

> For this product, does at least one in-stock variation exist?

For that case, this partial index is often better:

```sql
CREATE INDEX variations_in_stock_product_idx
ON variations (product_id)
WHERE stock > 0;
```

This only indexes rows where stock is greater than 0.

So Postgres searches a smaller index containing only in-stock variations.

Mental model:

```txt
(product_id, stock)
```

Good when you already know the product and want its stock/variations.

```txt
(stock, product_id)
```

Good when you start from stock and want all matching variations/products.

```txt
(product_id) WHERE stock > 0
```

Good for “does this product have at least one in-stock variation?”

---

# Partial Indexes

A partial index indexes only some rows.

Example:

```sql
CREATE INDEX variations_in_stock_product_idx
ON variations (product_id)
WHERE stock > 0;
```

This does not index every variation.

It only indexes variations where:

```sql
stock > 0
```

This is useful because many queries only care about in-stock products.

Instead of searching a huge index of all variations, Postgres can search a smaller index of only available ones.

Partial indexes are great when:

1. You frequently filter on the same condition
2. The condition removes a lot of rows
3. You do not need every row indexed for that query

---

# Indexes and Write Cost

Indexes are not free.

When you insert a row, Postgres has to:

```txt
1. Add the row to the real table
2. Add entries to every relevant index
3. Possibly rebalance or split index pages
```

When you update an indexed column, Postgres may have to update the index too.

When you delete a row, Postgres has to deal with the table row and the index entries.

So:

```txt
More indexes = faster reads, slower writes
```

Also:

```txt
Wider indexes = more storage and more maintenance
```

This index is lighter:

```sql
CREATE INDEX ON products (brand_id, name);
```

This index is heavier:

```sql
CREATE INDEX ON products (brand_id, name, price_cents, stock, created_at);
```

because it stores more values per index entry.

Do not index every column blindly.

Index the queries you actually use often.

---

# Corrected Mental Model

Original idea:

> Index builds a tree which sorts layer by layer, giving the preceding layer sort order priority. The more depth the index has, the faster getting that specific thing will be. However, this creates write complexity because the tree has to also be updated.

Corrected version:

> An index builds a sorted tree. For composite indexes, sorting happens left to right, so earlier columns matter most. This lets Postgres jump to a smaller section instead of scanning everything. More columns can make certain reads faster if they match the query, but they also make the index larger and writes slower because the tree has to be maintained whenever rows change.

Important correction:

> More depth does not mean faster.

A deeper tree usually means a larger index. Indexes are fast because each level eliminates many possibilities, not because the tree is deep.

B-trees stay wide and shallow on purpose.

---

# Practical Supabase Product Query Examples

## Basic Product Listing

Supabase:

```ts
supabase
  .from("products")
  .select("id, name, slug, min_price_cents, max_price_cents, thumbnail_url")
  .eq("brand_id", brandId)
  .order("name")
  .range(0, 23)
```

SQL concept:

```sql
SELECT id, name, slug, min_price_cents, max_price_cents, thumbnail_url
FROM products
WHERE brand_id = ?
ORDER BY name
LIMIT 24 OFFSET 0;
```

Good index:

```sql
CREATE INDEX products_brand_name_idx
ON products (brand_id, name);
```

Why it helps:

```txt
jump to brand
read products already sorted by name
stop after 24
```

## Product Listing Sorted By Price

Supabase:

```ts
supabase
  .from("products")
  .select("id, name, slug, min_price_cents, thumbnail_url")
  .eq("brand_id", brandId)
  .order("min_price_cents")
  .range(0, 23)
```

Good index:

```sql
CREATE INDEX products_brand_min_price_idx
ON products (brand_id, min_price_cents);
```

Why it helps:

```txt
jump to brand
read products already sorted by price
stop after 24
```

## Product Detail Page

Supabase:

```ts
supabase
  .from("products")
  .select("*")
  .eq("brand_id", brandId)
  .eq("slug", productSlug)
  .single()
```

Good index:

```sql
CREATE INDEX products_brand_slug_idx
ON products (brand_id, slug);
```

Even better if each slug is unique per brand:

```sql
CREATE UNIQUE INDEX products_brand_slug_unique_idx
ON products (brand_id, slug);
```

## Category Page

If you use a join table:

```txt
product_categories
- product_id
- category_id
```

Query concept:

```sql
SELECT p.*
FROM products p
JOIN product_categories pc ON pc.product_id = p.id
WHERE pc.category_id = ?
ORDER BY p.name
LIMIT 24;
```

Helpful indexes:

```sql
CREATE INDEX product_categories_category_product_idx
ON product_categories (category_id, product_id);
```

```sql
CREATE INDEX products_brand_name_idx
ON products (brand_id, name);
```

Depending on exact query shape, category + product lookup can benefit from the join table index.

## In-Stock Filter

Query concept:

```sql
SELECT *
FROM products p
WHERE p.brand_id = ?
AND EXISTS (
  SELECT 1
  FROM variations v
  WHERE v.product_id = p.id
  AND v.stock > 0
)
ORDER BY p.name
LIMIT 24;
```

Good index:

```sql
CREATE INDEX variations_in_stock_product_idx
ON variations (product_id)
WHERE stock > 0;
```

Why it helps:

```txt
only in-stock variations are indexed
Postgres can quickly check whether a product has at least one in-stock variation
```

---

# Recommended Indexes For Your Sunglasses Setup

These are reasonable starting indexes based on the product/search/category setup discussed.

## Products by Brand and Name

```sql
CREATE INDEX products_brand_name_idx
ON products (brand_id, name);
```

Use for:

```sql
WHERE brand_id = ...
ORDER BY name
LIMIT 24
```

## Products by Brand and Price

```sql
CREATE INDEX products_brand_min_price_idx
ON products (brand_id, min_price_cents);
```

Use for:

```sql
WHERE brand_id = ...
ORDER BY min_price_cents
LIMIT 24
```

## Product Detail by Brand and Slug

```sql
CREATE UNIQUE INDEX products_brand_slug_unique_idx
ON products (brand_id, slug);
```

Use for:

```sql
WHERE brand_id = ...
AND slug = ...
```

## Category Join Lookup

```sql
CREATE INDEX product_categories_category_product_idx
ON product_categories (category_id, product_id);
```

Use for:

```sql
WHERE category_id = ...
```

## Product-to-Category Reverse Lookup

```sql
CREATE INDEX product_categories_product_category_idx
ON product_categories (product_id, category_id);
```

Use when loading a product and needing its categories.

## In-Stock Variations

```sql
CREATE INDEX variations_in_stock_product_idx
ON variations (product_id)
WHERE stock > 0;
```

Use for:

```sql
WHERE product_id = ...
AND stock > 0
```

or `EXISTS` checks for in-stock products.

---

# What Not To Do

## Do Not Use `select("*")` Everywhere

This:

```ts
.select("*")
```

can be expensive if the table contains:

```txt
large descriptions
JSON attributes
metadata
long text arrays
fields the frontend does not need
```

Prefer selecting only what the UI needs.

For product cards:

```ts
.select("id, name, slug, min_price_cents, max_price_cents, sale_price_cents, thumbnail_url")
```

For product detail pages, `select("*")` is more acceptable because the user actually needs the full product.

## Do Not Add Random Indexes

Indexes should match real query patterns.

Bad reason to add an index:

> This column exists, so maybe I should index it.

Good reason to add an index:

> I frequently query `WHERE brand_id = ... ORDER BY name LIMIT 24`, so I should index `(brand_id, name)`.

## Do Not Assume Pagination Alone Fixes Bad Queries

Pagination reduces what comes back.

Indexes reduce how hard it is to find the rows.

You usually want both.

---

# Best Mental Models

## Pagination

```txt
Pagination limits what comes back.
```

But:

```txt
Indexes determine how efficiently Postgres finds what comes back.
```

## Index

```txt
The table stores the real rows.
The index stores sorted shortcuts pointing to those rows.
```

## Composite Index

```txt
Composite indexes are sorted left to right.
Earlier columns matter most.
```

## Brand + Name Index

```txt
(brand_id, name)
=
group/sort by brand first,
then sort by name inside each brand.
```

## Indexed Brand Search

```txt
Without index:
check every row.

With index:
jump to the brand section,
read matching entries,
stop when the brand changes.
```

## Reads vs Writes

```txt
Indexes make reads faster.
Indexes make writes slower.
```

---

# Final Summary

A Supabase query is fast or slow based on how much work Postgres has to do.

Pagination helps because it returns fewer rows and reduces response size, JSON work, browser memory, and rendering work.

Indexes help because they let Postgres avoid scanning the whole table.

A single-column index like:

```sql
CREATE INDEX ON products (brand_id);
```

helps filter by brand.

A composite index like:

```sql
CREATE INDEX ON products (brand_id, name);
```

helps filter by brand and return products already sorted by name.

Composite indexes work from left to right, so `(brand_id, name)` helps brand queries, but `(name, brand_id)` is not ideal if your query starts with brand.

For your product listing pages, the most important idea is:

```txt
WHERE brand_id = ...
ORDER BY name
LIMIT 24
```

matches:

```sql
CREATE INDEX ON products (brand_id, name);
```

That allows Postgres to jump to the right brand, read rows in the correct order, and stop after 24.

That is the real power of indexes with pagination.

