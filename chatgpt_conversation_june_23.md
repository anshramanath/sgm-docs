# Product Detail Endpoint, IDs, Slugs, React Keys, and Variation Indexing Notes (June 23, 2026)

## 1. Product detail endpoint

Current public product detail endpoint shape:

```ts
const brandSlug = req.nextUrl.searchParams.get("brandSlug");
const slug = req.nextUrl.searchParams.get("slug");

const { data: product, error } = await supabase
  .from("products")
  .select(`
    name, slug, sku, description, summary, attributes, featured,
    sale, min_price_cents, max_price_cents, sale_price_cents,
    variations(sku, attribute, sale, regular_price_cents, sale_price_cents,
      variation_images(src, name, sort_order)
    ),
    product_images(src, name, sort_order),
    product_description_images(
      description_images(src, name)
    )
  `)
  .eq("slug", slug)
  .eq("brand_slug", brandSlug)
  .single();
```

This endpoint starts from the `products` table.

The lookup path is:

```txt
brandSlug + slug
↓
find one product row
↓
use product.id internally
↓
fetch product_images
↓
fetch variations
↓
fetch variation_images
↓
fetch description_images
```

The public API uses `brandSlug + slug`, but internally the database relationships still work through IDs.

---

## 2. Should the endpoint use `id` instead of `slug`?

Technically, `id` is the fastest and cleanest database lookup because it is the primary key.

Example:

```ts
.eq("id", productId)
.single()
```

Postgres automatically indexes primary keys, so this is naturally fast:

```txt
products.id = unique + indexed automatically
```

However, for a public storefront/product page, `brandSlug + slug` is usually better because it creates readable, SEO-friendly URLs.

Example public URL:

```txt
/bikershades/product/foam-padded-sunglasses
```

is better than:

```txt
/product/7c89aef8-abbc-49b6-9d6e-e97bacb6c89f
```

### Best rule

```txt
Public product route:
  use brandSlug + slug

Internal/admin/cart/database operations:
  use id
```

---

## 3. Make `brandSlug + slug` fast

Since product slugs are not globally unique across all brands, the correct constraint is:

```sql
ALTER TABLE products
ADD CONSTRAINT products_brand_slug_unique
UNIQUE (brand_slug, slug);
```

This creates a unique B-tree index on the pair:

```txt
(brand_slug, slug)
```

That means this query is fast:

```sql
WHERE brand_slug = ?
AND slug = ?
```

Postgres searches for the exact combined key:

```txt
(bikershades, foam-padded)
```

It does not need to scan every product in the brand if both values are provided.

---

## 4. Why not rely on slug alone?

If slugs are only unique inside a brand, this is unsafe:

```ts
.eq("slug", slug)
.single()
```

Because the same slug could exist in multiple brands:

```txt
bikershades / aviator
prosport    / aviator
monster     / aviator
```

So the endpoint should stay:

```ts
.eq("brand_slug", brandSlug)
.eq("slug", slug)
.single()
```

with:

```sql
UNIQUE (brand_slug, slug)
```

---

## 5. Can a composite unique index be used without both values?

For this index:

```sql
UNIQUE (brand_slug, slug)
```

The index is sorted like:

```txt
brand_slug first
  slug inside each brand
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

Not good:

```sql
WHERE slug = ?
```

because `slug` is the second column and `brand_slug` is missing.

### Rule

```txt
Composite indexes work efficiently from left to right.
You cannot skip the first column and expect the second column to be fast.
```

---

## 6. React list keys

In React lists, using the Supabase/Postgres `id` as the `key` is generally the best choice.

Example:

```tsx
{products.map((product) => (
  <ProductCard key={product.id} product={product} />
))}
```

React wants a key that is:

```txt
unique
stable
not generated during render
not based on array position
```

Database IDs fit that perfectly.

### Avoid array indexes

Avoid:

```tsx
{products.map((product, index) => (
  <ProductCard key={index} product={product} />
))}
```

because if the list order changes, filters change, or items are inserted/deleted, React can confuse component state between rows.

### Slugs can work, but IDs are safer

This can work if slug is unique in that list:

```tsx
<ProductCard key={product.slug} product={product} />
```

But IDs are safer because slugs can be brand-scoped or can change.

### Rule

```txt
Best key:
  database id

Good fallback:
  stable unique slug

Avoid:
  array index
  random UUID generated during render
```

---

## 7. Should variations include `id` in the API?

For React rendering variation lists, it is usually helpful to include `variation.id`.

Example:

```tsx
{product.variations.map((variation) => (
  <VariationOption key={variation.id} variation={variation} />
))}
```

If the API does not expose `variation.id`, then a stable variation slug/SKU can work if it is unique within the product.

But from a frontend stability perspective, returning variation IDs is usually clean.

---

## 8. Should `brand_slug` be added to `variations`?

For the product detail endpoint: **no**.

Do not add `brand_slug` to `variations` just for this route.

The endpoint already does:

```txt
brandSlug + slug
↓
products table
↓
find product.id
↓
variations.product_id
```

So the useful variation index is:

```sql
CREATE INDEX variations_product_idx
ON variations (product_id);
```

You do not need:

```sql
CREATE INDEX variations_brand_product_idx
ON variations (brand_slug, product_id);
```

for this endpoint.

---

## 9. Why `brand_slug` on variations does not help this endpoint

If you added:

```sql
CREATE INDEX variations_brand_product_idx
ON variations (brand_slug, product_id);
```

the index would be sorted like:

```txt
brand_slug first
  product_id inside each brand
```

That helps queries like:

```sql
WHERE brand_slug = ?
AND product_id = ?
```

or:

```sql
WHERE brand_slug = ?
```

But the product detail endpoint does not start from variations by brand.

It starts with one product:

```sql
WHERE products.brand_slug = ?
AND products.slug = ?
```

Then it fetches child rows using:

```sql
WHERE variations.product_id = ?
```

So the best index starts with `product_id`:

```sql
CREATE INDEX variations_product_idx
ON variations (product_id);
```

### Rule

```txt
Index should start with the value your query actually filters by first.
```

For this route, the known child lookup value is:

```txt
product_id
```

So the index should be:

```txt
variations(product_id)
```

not:

```txt
variations(brand_slug, product_id)
```

---

## 10. But wouldn't `brand_slug` sort product IDs by brand?

Yes.

An index on:

```sql
(brand_slug, product_id)
```

would sort like:

```txt
bikershades
  product_id 1
  product_id 2
  product_id 3

prosport
  product_id 4
  product_id 5
```

That is useful if your query starts with:

```sql
WHERE brand_slug = 'bikershades'
```

But for the detail endpoint, the query already has one exact product ID after finding the product.

So grouping variation rows by brand first does not help the lookup:

```sql
WHERE product_id = ?
```

In fact, if the index is `(brand_slug, product_id)`, then `product_id` is second, so the index is not ideal unless `brand_slug` is also provided.

---

## 11. Product ID is already brand-scoped through the product table

A `product_id` points to exactly one row in `products`.

That product row already has:

```txt
brand_slug
slug
name
sku
```

So this relationship already exists:

```txt
variations.product_id
↓
products.id
↓
products.brand_slug
```

Adding `brand_slug` directly to `variations` would duplicate data.

That is denormalization.

Denormalization can sometimes be useful for performance, but only when there is a real query pattern that needs it.

For this endpoint, there is no need.

---

## 12. When would `brand_slug` on variations make sense?

It might make sense if you had a route that directly queried variations by brand, such as:

```sql
SELECT *
FROM variations
WHERE brand_slug = ?
```

Examples:

```txt
brand-wide inventory dashboard
brand-wide variation search
admin stock table where variations are the root resource
bulk export of all variations for a brand
```

Even then, it is usually better to first try a join through `products`:

```sql
SELECT v.*
FROM variations v
JOIN products p ON p.id = v.product_id
WHERE p.brand_slug = ?;
```

Then, if performance becomes bad at scale, you could consider denormalizing `brand_slug` onto `variations`.

Do not add duplicated brand fields early unless a real query needs them.

---

## 13. Correct index map for the product detail endpoint

### Product lookup

```sql
ALTER TABLE products
ADD CONSTRAINT products_brand_slug_unique
UNIQUE (brand_slug, slug);
```

### Product → variations

```sql
CREATE INDEX variations_product_idx
ON variations (product_id);
```

### Product → product images

```sql
CREATE INDEX product_images_product_idx
ON product_images (product_id);
```

### Variation → variation images

```sql
CREATE INDEX variation_images_variation_idx
ON variation_images (variation_id);
```

### Product → description image join table

```sql
CREATE INDEX product_description_images_product_idx
ON product_description_images (product_id);
```

If there is also a direct lookup from the join table to description images, the description image primary key should already be indexed if it is the `id`.

---

## 14. If database-level image ordering is added later

Right now the endpoint sorts images in JavaScript:

```ts
.sort((a, b) => a.sort_order - b.sort_order)
```

That is okay.

If you later move ordering into the database, these indexes may help:

```sql
CREATE INDEX product_images_product_sort_idx
ON product_images (product_id, sort_order);

CREATE INDEX variation_images_variation_sort_idx
ON variation_images (variation_id, sort_order);
```

But if JavaScript sorting is fine for now, the simpler foreign-key indexes are enough.

---

## 15. Final clean mental model

```txt
Use slug + brandSlug for public product pages because URLs are readable and SEO-friendly.

Make slug + brandSlug fast with a composite unique constraint.

Use id internally when you already have the id.

Use database ids as React keys.

Do not duplicate brand_slug into variations for this endpoint.

Child tables should point to products.id.

Index child-table foreign keys.

For variations under one product, the useful index is variations(product_id), not variations(brand_slug, product_id).
```

---

## 16. Best current setup

For this route, the best setup is:

```sql
ALTER TABLE products
ADD CONSTRAINT products_brand_slug_unique
UNIQUE (brand_slug, slug);

CREATE INDEX variations_product_idx
ON variations (product_id);

CREATE INDEX product_images_product_idx
ON product_images (product_id);

CREATE INDEX variation_images_variation_idx
ON variation_images (variation_id);

CREATE INDEX product_description_images_product_idx
ON product_description_images (product_id);
```

And the endpoint can stay:

```ts
.eq("brand_slug", brandSlug)
.eq("slug", slug)
.single()
```

This is a good balance of:

```txt
SEO-friendly public URLs
fast indexed product lookup
clean normalized schema
simple child-table joins
stable frontend rendering
```
