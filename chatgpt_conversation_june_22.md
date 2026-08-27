# Supabase / Postgres Indexing, Joins, Pagination, and Next.js Caching Notes (June 22, 2026)

## 1. Pagination and database work

Pagination does **not** mean the database selects everything and then the frontend only receives a smaller amount.

A Supabase query like:

```ts
supabase
  .from("products")
  .select("*")
  .eq("brand_slug", brandSlug)
  .range(0, 19)
```

becomes conceptually similar to:

```sql
SELECT *
FROM products
WHERE brand_slug = ?
LIMIT 20 OFFSET 0;
```

The database does not normally load every product into memory first. Postgres uses its query planner to decide the cheapest way to find the requested rows.

However, pagination only becomes truly fast when the database has indexes that help it find the correct rows quickly.

### No pagination mainly hurts because of:

- More network response data
- More JSON serialization/parsing
- More browser memory usage
- More React rendering work
- Potentially more database work, because the database cannot stop early

### Pagination with no useful index

If a query has to filter/sort without an index, Postgres may still scan or sort many rows before returning 20.

### Pagination with useful index

If the query and index match, Postgres can jump to the correct section and stop early.

Example:

```sql
CREATE INDEX products_brand_name_idx
ON products (brand_slug, name);
```

This helps:

```sql
WHERE brand_slug = ?
ORDER BY name
LIMIT 20;
```

because the rows are already grouped by brand and ordered by name inside each brand.

---

## 2. Why `ORDER BY` matters for pagination

Pagination without ordering is unstable.

```ts
.range(0, 19)
```

without `order()` means:

> Give me any 20 matching rows.

Then page 2:

```ts
.range(20, 39)
```

means:

> Give me another 20 rows, but the database has no guaranteed stable order.

This can cause duplicates or skipped products across pages if the database chooses a different plan or the data changes.

So always use an explicit order when paginating:

```ts
.order("id", { ascending: true })
.range(from, to)
```

or:

```ts
.order("name", { ascending: true })
.range(from, to)
```

### `ORDER BY` and indexes

An `ORDER BY` is required for stable pagination, but the order column does not have to be indexed for correctness.

However, adding the order column to the index can make pagination faster because Postgres can read rows already ordered and grab the first 20.

Example:

```sql
CREATE INDEX product_categories_category_product_idx
ON product_categories (category_id, product_id);
```

This helps:

```ts
.eq("category_id", categoryId)
.order("product_id")
.range(from, to)
```

because:

```txt
category_id = find the category section
product_id = already ordered inside that category
range = grab the first 20
```

---

## 3. What indexes are

An index is a separate data structure stored beside the table.

The table may be physically stored in any order:

```txt
products table
row 1: proSPORT, "Baseball Glasses"
row 2: BikerShades, "Aviator"
row 3: BikerShades, "Foam Padded"
```

An index stores selected columns in sorted order with pointers back to the real rows.

Example:

```sql
CREATE INDEX products_brand_name_idx
ON products (brand_slug, name);
```

The index is conceptually:

```txt
brand_slug       name             row pointer
bikershades      Aviator          row 2
bikershades      Foam Padded      row 3
prosport         Baseball Glasses row 1
```

The table itself is not rearranged. The index is just a sorted shortcut.

---

## 4. B-tree indexes

Postgres usually uses B-tree indexes for normal indexes.

A B-tree is not usually a simple binary tree. It is a wide, shallow tree.

Postgres can search the tree quickly to find a value or range.

For:

```sql
CREATE INDEX products_brand_slug_name_idx
ON products (brand_slug, name);
```

Postgres can quickly find:

```sql
WHERE brand_slug = 'bikershades'
ORDER BY name
```

because the index is sorted by:

```txt
brand_slug first
  name inside each brand
```

---

## 5. Composite indexes and leftmost prefix

A composite index has multiple columns:

```sql
CREATE INDEX products_brand_category_name_idx
ON products (brand_slug, category_id, name);
```

It is sorted like:

```txt
brand_slug first
  category_id inside each brand
    name inside each brand/category
```

This can help:

```sql
WHERE brand_slug = ?
```

and:

```sql
WHERE brand_slug = ?
AND category_id = ?
```

and:

```sql
WHERE brand_slug = ?
AND category_id = ?
ORDER BY name
```

But it is not very helpful for:

```sql
WHERE category_id = ?
```

because that skips the first column.

### Rule

```txt
You can use a composite index efficiently from left to right.
You cannot skip the first column and expect the second column to be fast.
```

---

## 6. Indexes for groups vs exact lookup

Indexes help both grouped values and unique/exact values.

### Non-unique column

```sql
CREATE INDEX products_brand_idx
ON products (brand_slug);
```

This helps find a group:

```sql
WHERE brand_slug = 'bikershades'
```

Postgres jumps to the `bikershades` section and reads all matching rows.

### Unique/exact lookup

```sql
CREATE UNIQUE INDEX products_brand_slug_unique_idx
ON products (brand_slug, slug);
```

This helps find one exact row:

```sql
WHERE brand_slug = 'bikershades'
AND slug = 'foam-padded'
```

Postgres searches for the exact combined key:

```txt
(bikershades, foam-padded)
```

It does not scan all products in the brand if both values are provided.

---

## 7. Unique constraints vs normal indexes

A normal index:

```sql
CREATE INDEX products_slug_idx
ON products (slug);
```

means:

> Make slug searchable faster.

A unique constraint:

```sql
ALTER TABLE products
ADD CONSTRAINT products_brand_slug_unique
UNIQUE (brand_slug, slug);
```

means:

> Prevent duplicate brand/slug pairs.

Postgres enforces uniqueness using a unique B-tree index.

So a unique constraint gives you:

```txt
speed structure + no-duplicate rule
```

### Important

Uniqueness alone in your head does not make lookup fast.

The database needs an actual unique constraint or index.

### Slug rules

If `slug` is globally unique:

```sql
ALTER TABLE products
ADD CONSTRAINT products_slug_unique
UNIQUE (slug);
```

If `slug` is only unique inside a brand:

```sql
ALTER TABLE products
ADD CONSTRAINT products_brand_slug_unique
UNIQUE (brand_slug, slug);
```

For this product detail endpoint:

```ts
.eq("brand_slug", brandSlug)
.eq("slug", slug)
.single()
```

use:

```sql
ALTER TABLE products
ADD CONSTRAINT products_brand_slug_unique
UNIQUE (brand_slug, slug);
```

because the lookup uses both values.

---

## 8. Can you use a composite unique index without both values?

Yes, but only efficiently from the left side.

For:

```sql
UNIQUE (brand_slug, slug)
```

Good:

```sql
WHERE brand_slug = ?
```

Great:

```sql
WHERE brand_slug = ?
AND slug = ?
```

Not great:

```sql
WHERE slug = ?
```

because `slug` is second and `brand_slug` is missing.

---

## 9. Why a brand-only index is not enough for detail lookup

If you only had:

```sql
CREATE INDEX products_brand_idx
ON products (brand_slug);
```

then this query:

```sql
WHERE brand_slug = ?
AND slug = ?
```

could find all products for that brand, then scan within that brand to find the slug.

But with:

```sql
UNIQUE (brand_slug, slug)
```

Postgres can search for the exact combined key.

```txt
brand-only index:
  jump to brand group, then scan inside group

brand + slug index:
  jump/search directly to exact pair
```

---

## 10. Joins and indexes

Indexes do not apply to a final temporary joined table. They apply to the base tables used to build the joined result.

Example query:

```sql
SELECT p.*
FROM product_categories pc
JOIN products p ON p.id = pc.product_id
WHERE pc.category_id = ?;
```

Postgres may use:

```sql
product_categories(category_id, product_id)
```

to find matching product IDs.

Then it uses:

```sql
products(id)
```

to fetch actual product rows.

There is no permanent joined table with its own index.

---

## 11. Primary key vs foreign key indexes

Primary keys are automatically indexed.

If you define:

```sql
id uuid PRIMARY KEY
```

Postgres automatically creates a unique index on `id`.

Foreign keys are **not necessarily indexed automatically**.

Example:

```txt
variations.product_id → products.id
```

`products.id` is indexed because it is the primary key.

But `variations.product_id` needs its own index if you often query:

```sql
WHERE product_id = ?
```

### Rule

```txt
Child → parent:
  fast because parent primary key is indexed.

Parent → children:
  needs an index on the child foreign key.
```

---

## 12. Join table: `product_categories`

The join table probably looks like:

```txt
product_categories

product_id   → foreign key to products.id
category_id  → foreign key to categories.id
```

This means:

```txt
Product P1 belongs to Category C1
Product P1 belongs to Category C2
Product P2 belongs to Category C1
```

For category pages, the query direction is:

```txt
category_id → product_categories rows → product_ids → products rows
```

So the important index is:

```sql
CREATE INDEX product_categories_category_product_idx
ON product_categories (category_id, product_id);
```

### Why include `product_id`?

Because `category_id` is the search key, and `product_id` is the value needed next for the join.

With only:

```sql
CREATE INDEX ON product_categories (category_id);
```

Postgres can find matching rows, but may need to visit the table rows to read `product_id`.

With:

```sql
CREATE INDEX ON product_categories (category_id, product_id);
```

the index itself contains:

```txt
category_id → product_id values
```

So it can sometimes read product IDs directly from the index.

This is a covering-style benefit.

---

## 13. Duplicate rows in join table

A product can belong to multiple categories. This is normal:

```txt
product_id   category_id
P1           sunglasses
P1           polarized
P1           mens
```

But this is usually bad:

```txt
product_id   category_id
P1           sunglasses
P1           sunglasses
```

That is the same relationship inserted twice.

To prevent this:

```sql
ALTER TABLE product_categories
ADD CONSTRAINT product_categories_category_product_unique
UNIQUE (category_id, product_id);
```

This does not stop a product from being in multiple categories. It only stops the same product/category pair from being duplicated.

Before adding it, check for duplicates:

```sql
SELECT category_id, product_id, COUNT(*)
FROM product_categories
GROUP BY category_id, product_id
HAVING COUNT(*) > 1;
```

If this returns no rows, add the constraint.

---

## 14. Starting from `products` vs starting from `product_categories`

Original category route shape:

```ts
supabase
  .from("products")
  .select(`
    id,
    name,
    slug,
    product_categories!inner(category_id)
  `)
  .eq("brand_slug", brandSlug)
  .eq("product_categories.category_id", categoryId)
```

Even though this starts with `.from("products")`, Postgres can internally choose to start from `product_categories` if that is cheaper.

The planner may choose:

```txt
category_id
↓
product_categories(category_id, product_id)
↓
product_ids
↓
products(id)
```

So you do not always need to rewrite the route just to get the join-table-first plan.

However, starting from `product_categories` makes the query shape more directly match the route data:

```ts
supabase
  .from("product_categories")
  .select(`
    products!inner(...)
  `)
  .eq("category_id", categoryId)
  .eq("products.brand_slug", brandSlug)
```

This says:

```txt
categoryId from route
↓
find product IDs
↓
join products
```

Both can be valid. The planner chooses the cheapest path based on indexes and table statistics.

---

## 15. Filtering joined table columns in Supabase

If the root table is:

```ts
.from("products")
```

then product columns do not need a prefix:

```ts
.eq("brand_slug", brandSlug)
```

Joined table columns need a prefix:

```ts
.eq("product_categories.category_id", categoryId)
```

If the root table is:

```ts
.from("product_categories")
```

then join table columns do not need a prefix:

```ts
.eq("category_id", categoryId)
```

Product columns need a prefix:

```ts
.eq("products.brand_slug", brandSlug)
.eq("products.sale", true)
.gte("products.min_price_cents", minPrice)
```

---

## 16. Category endpoint after rewrite

Your rewritten category endpoint uses:

```ts
.from("product_categories")
.select(`
  products!inner(
    id, name, slug, featured, sale, min_price_cents, max_price_cents, sale_price_cents,
    product_images!inner(src, name, sort_order),
    variations(attribute,
      variation_images(src, name, sort_order)
    )
  )
`, { count: "exact" })
.eq("category_id", categoryId)
.eq("products.brand_slug", brandSlug)
```

This is structurally reasonable.

You added:

```ts
.order("product_id", { ascending: true })
.range(from, to)
```

That gives stable pagination.

The matching index is:

```sql
CREATE INDEX product_categories_category_product_idx
ON product_categories (category_id, product_id);
```

This index supports:

```txt
WHERE category_id = ?
ORDER BY product_id
LIMIT/OFFSET
```

It also provides the `product_id` values needed for the join.

### Note

Ordering by `product_id` gives stable pagination, but not necessarily a user-friendly product order. It may look random because product IDs are not product names.

If you want storefront-friendly sorting, order by product name instead. That may be more awkward from a `product_categories` root.

---

## 17. Sale endpoint

Your sale endpoint uses:

```ts
.from("products")
.eq("brand_slug", brandSlug)
.eq("sale", true)
.order("id", { ascending: true })
.range(from, to)
```

This is valid.

`id` is already indexed because it is the primary key.

But a better full query-pattern index is:

```sql
CREATE INDEX products_sale_brand_id_idx
ON products (brand_slug, id)
WHERE sale = true;
```

This supports:

```txt
brand_slug = ?
sale = true
ORDER BY id
LIMIT/OFFSET
```

The partial index only stores sale products, which makes it smaller.

---

## 18. Price filters

Your filters:

```ts
.gte("min_price_cents", minPrice)
.lte("min_price_cents", maxPrice)
```

can benefit from an index like:

```sql
CREATE INDEX products_brand_min_price_idx
ON products (brand_slug, min_price_cents);
```

For sale page price filtering, you could also use:

```sql
CREATE INDEX products_sale_brand_min_price_idx
ON products (brand_slug, min_price_cents)
WHERE sale = true;
```

However, if the route orders by `id`, then price filtering and `id` ordering cannot both be perfectly optimized by the same simple index.

Start simple. Add price-specific indexes only if price-filter pages are slow.

---

## 19. Product detail endpoint

Your product detail endpoint does:

```ts
.from("products")
.select(`
  id, name, sku, description, summary, attributes, featured,
  sale, min_price_cents, max_price_cents, sale_price_cents,
  variations(...),
  product_images(...),
  product_description_images(...)
`)
.eq("slug", slug)
.eq("brand_slug", brandSlug)
.single()
```

Since slug is not globally unique, use:

```sql
ALTER TABLE products
ADD CONSTRAINT products_brand_slug_unique
UNIQUE (brand_slug, slug);
```

Then child-table indexes:

```sql
CREATE INDEX variations_product_idx
ON variations (product_id);

CREATE INDEX product_images_product_idx
ON product_images (product_id);

CREATE INDEX product_description_images_product_idx
ON product_description_images (product_id);

CREATE INDEX variation_images_variation_idx
ON variation_images (variation_id);
```

If you later make the database order nested images by `sort_order`, use:

```sql
CREATE INDEX product_images_product_sort_idx
ON product_images (product_id, sort_order);

CREATE INDEX variation_images_variation_sort_idx
ON variation_images (variation_id, sort_order);
```

But if sorting remains in JS, the plain foreign-key indexes are enough for now.

---

## 20. Nested data overfetching

Your listing endpoints fetch:

```txt
20 products
  all product images
  all variations
    all variation images
```

Then JS keeps only:

```txt
first product image
unique color swatches
first variation image per color
```

That means the product filtering is already happening in the database, but nested data may still be overfetched.

This is not urgent unless performance becomes an issue.

A later advanced optimization would be an RPC/view that returns exactly the product-card shape.

For now, limiting the scope to indexes is reasonable.

---

## 21. Next.js / Vercel caching

Your frontend server uses:

```ts
fetch(url.toString(), { next: { revalidate: 60 } })
```

This is Next.js server-side Data Cache behavior.

It means:

```txt
Next/Vercel frontend server caches backend API responses for 60 seconds.
```

This is not the same as normal browser cache.

### Cache layers

```txt
Browser
↓
CDN / edge cache
↓
Next frontend server
↓
Next Data Cache from revalidate
↓
Your backend API
↓
Supabase
```

### CDN cache

CDN cache means:

> The response/assets sent to the browser are stored closer to the user.

### Revalidate/Data Cache

`next: { revalidate: 60 }` means:

> The Next/Vercel server can reuse a saved fetch result instead of calling your backend/Supabase again.

So:

```txt
CDN cache:
  cached HTTP response near the user

Next Data Cache:
  cached server-side fetch result used by Next

Browser/router cache:
  cached page navigation/assets inside the browser
```

---

## 22. Final recommended index set

### Product detail lookup

```sql
ALTER TABLE products
ADD CONSTRAINT products_brand_slug_unique
UNIQUE (brand_slug, slug);
```

### Category listing

```sql
CREATE INDEX product_categories_category_product_idx
ON product_categories (category_id, product_id);
```

Optional safety if no duplicates exist:

```sql
ALTER TABLE product_categories
ADD CONSTRAINT product_categories_category_product_unique
UNIQUE (category_id, product_id);
```

If you add the unique constraint, it already creates a useful index for `(category_id, product_id)`, so you may not need the separate non-unique index.

### Sale listing

```sql
CREATE INDEX products_sale_brand_id_idx
ON products (brand_slug, id)
WHERE sale = true;
```

### Brand/product listing by name

```sql
CREATE INDEX products_brand_slug_name_idx
ON products (brand_slug, name);
```

### Price filtering

Optional:

```sql
CREATE INDEX products_brand_min_price_idx
ON products (brand_slug, min_price_cents);
```

Optional sale-specific version:

```sql
CREATE INDEX products_sale_brand_min_price_idx
ON products (brand_slug, min_price_cents)
WHERE sale = true;
```

### Product detail nested children

```sql
CREATE INDEX variations_product_idx
ON variations (product_id);

CREATE INDEX product_images_product_idx
ON product_images (product_id);

CREATE INDEX product_description_images_product_idx
ON product_description_images (product_id);

CREATE INDEX variation_images_variation_idx
ON variation_images (variation_id);
```

### If DB-level sort by image order is added later

```sql
CREATE INDEX product_images_product_sort_idx
ON product_images (product_id, sort_order);

CREATE INDEX variation_images_variation_sort_idx
ON variation_images (variation_id, sort_order);
```

---

## 23. Final mental model

```txt
Indexes help Postgres find rows, join rows, and sometimes return rows already ordered.

Primary keys are indexed automatically.

Foreign keys are not automatically indexed, so add indexes on child-table foreign keys when querying parent → children.

Composite indexes work left to right.

Unique constraints create unique indexes.

ORDER BY is required for stable pagination.

ORDER BY + matching index makes pagination faster.

Indexes do not index the final joined result. They help the planner build the joined result efficiently from base tables.
```

