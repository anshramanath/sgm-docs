# What I Built & Learned (June 22, 2026)

## What I Built

A Next.js API-only server for a multi-brand sunglasses e-commerce platform. Multiple storefronts share one codebase and database, scoped by `brand_slug`. No frontend — pure API.

### Endpoints

| Method | Path | What it does |
|--------|------|--------------|
| GET | `/api/public/brands` | All brands |
| GET | `/api/public/categories` | Full recursive category tree, sorted |
| GET | `/api/public/products` | Paginated products by category with color swatches |
| GET | `/api/public/sale` | Paginated sale products |
| GET | `/api/public/item` | Full product detail with all variations and images |
| GET | `/api/public/search` | Case-insensitive name search, up to 6 results |
| POST | `/api/public/validate-cart` | Check cart SKUs still exist before checkout |
| GET | `/api/user/cart` | Fetch user cart |
| PUT | `/api/user/cart` | Replace user cart (full sync) |
| GET | `/api/user/bookmarks` | Fetch user bookmarks |
| PUT | `/api/user/bookmarks` | Replace user bookmarks (full sync) |
| GET | `/api/user/orders` | Order history with shipping address and line items |
| POST | `/api/user/checkout` | Create Stripe checkout session |
| POST | `/api/webhooks/stripe` | Handle `checkout.session.completed` |

### Database Design

13 tables across 3 domains:

**Catalog** — `brands`, `categories` (self-referential tree via `parent_id`), `products`, `product_categories` (many-to-many junction), `variations`, `product_images`, `variation_images`, `description_images`, `product_description_images` (many-to-many junction)

**Commerce** — `orders`, `order_items`

**User** — `cart_items`, `bookmarks`

Key design decisions:
- Products belong to categories through a junction table (`product_categories`) so one product can appear in multiple categories
- `attributes` on products is `jsonb` — stores the full color/size option menu for the product page
- `attribute` on variations is `jsonb` — stores the specific combination for that SKU (e.g. Gloss Black + Standard)
- `description_images` is a shared pool per brand; `product_description_images` links products to them
- `total_sales` is incremented atomically via Postgres RPC functions, not in application code

---

## What I Learned

### Supabase Query Builder

**Two clients serve different purposes.** The admin client uses the service role key and bypasses RLS — used for all public and webhook routes. The user client is built from the Bearer token in the request header and respects RLS — used for cart, bookmarks, orders, and checkout.

**Embedding related tables.** Supabase lets you select related rows inline using the foreign key relationship name:
```ts
supabase.from("products").select(`
  id, name,
  variations(id, sku, attribute),
  product_images(src, name, sort_order)
`)
```
This returns nested arrays on each row instead of requiring a JOIN.

**`!inner` makes the join exclusive.** Without `!inner`, products with no images still return (with an empty array). With `!inner`, only products that have at least one matching row come back — equivalent to `INNER JOIN`.

**Filtering on embedded tables.** You can filter on nested columns using dot notation:
```ts
.eq("products.brand_slug", brandSlug)
```
This only works on the embedded table, not the root.

**`referencedTable` orders the nested result, not the root.** To order the main result set when starting from a junction table, order by a column on that table itself:
```ts
// This orders junction rows, which effectively orders products
.order("product_id", { ascending: true })

// This would order the embedded products array, not the root
.order("slug", { referencedTable: "products" })
```

**Starting from a junction table.** For `products` filtered by category, starting from `product_categories` instead of `products` gives the DB a smaller starting set:
```ts
supabase.from("product_categories")
  .select(`products!inner(id, name, ...)`)
  .eq("category_id", categoryId)
```
The tradeoff: `r.products` comes back as an array (Supabase types it that way for many-to-many), so you need `flatMap` to unwrap it:
```ts
data.flatMap((r) => r.products ?? [])
```

**Many-to-many through a junction always needs `flatMap`.** Same pattern for `product_description_images → description_images`:
```ts
product.product_description_images.flatMap((r) => r.description_images)
```

### Pagination

Supabase uses `.range(from, to)` (inclusive on both ends) with `{ count: "exact" }` to get total count alongside the page:
```ts
const from = (page - 1) * size;
const to = from + size - 1;
const { data, count } = await q.range(from, to);
```

**ORDER BY is required for consistent pagination.** Without it, Postgres returns rows in heap order which is non-deterministic — the same page number can return different products on repeated requests. Always add an explicit order before paginating.

### Color Deduplication

A product with 4 colors and 2 sizes has 8 variations. For a product listing card, you only want to show 4 color swatches — one per unique color, each with its first image.

The pattern: `reduce` over variations, tracking seen color slugs in a `Set`, only pushing when the color is new:
```ts
const seen = new Set<string>();
const variations = product.variations.reduce((acc, v) => {
  const color = v.attribute.find((a) => a.name === "color");
  if (!color || seen.has(color.slug)) return acc;
  seen.add(color.slug);
  const firstImage = v.variation_images.sort(...)[0];
  acc.push({ option: color.option, slug: color.slug, value: color.value, imageSrc: firstImage?.src ?? null, imageName: firstImage?.name ?? null });
  return acc;
}, []);
```

`reduce` is used instead of `map` because the output array is shorter than the input — `map` always produces the same length.

Once you're past the `if (!color || seen.has(color.slug)) return acc` guard, `color` and `color.value` are guaranteed to exist, so no nullish fallback needed on the push.

### Response Shape

All responses share a wrapper:
```ts
// ok
{ success: true, data: { ... } }

// err
{ success: false, error: "Message" }
```

Implemented in two functions in `lib/api.ts`:
```ts
export function ok(data: unknown, status: number) {
  return NextResponse.json({ success: true, data }, { status });
}
export function err(message: string, status: number) {
  return NextResponse.json({ success: false, error: message }, { status });
}
```

**When `data` is the object vs. when it's wrapped.** Item returns one product, so `data` IS the product directly — `ok(item, 200)`. Products and sale return multiple keys (products array + pagination meta), so they need an object wrapper — `ok({ products, page, totalPages, ... }, 200)`.

### Stripe Integration

**Checkout session creation.** Stripe collects shipping — no need to build an address form on the frontend. Line items include product name + SKU in a parseable format so the webhook can recover them:
```ts
name: `${item.name} (${item.sku})`
```

**Idempotency key.** The same cart submitted twice shouldn't create two sessions. Key is derived from `userId + hash(cart) + orderCount`. If the user adds an item, the hash changes and a new session is created.

**Webhook handler.** Stripe sends `checkout.session.completed`. The handler:
1. Verifies the `stripe-signature` header
2. Checks for an existing order by `stripe_session_id` (idempotency)
3. Fetches line items from Stripe (not from the session object — they're paginated separately)
4. Recovers SKUs by parsing the description string
5. Inserts `orders` + `order_items`
6. Increments `total_sales` via Postgres RPC

**SKU recovery pattern.** Stripe stores the product name as the line item description. Parsing:
```ts
const start = desc.lastIndexOf("(");
const sku = desc.slice(start + 1, -1); // between last ( and )
const name = desc.slice(0, start - 1); // before " ("
```

### Category Tree

Fetched flat, assembled in memory using a `nodeMap` (object keyed by id). Two passes: first build all nodes, then wire parent-child links. Recursive sort after assembly:
```ts
function sortTree(nodes: CategoryNode[]) {
  nodes.sort((a, b) => a.sortOrder - b.sortOrder);
  for (const node of nodes) {
    if (node.children) sortTree(node.children);
  }
}
```

### RLS (Row Level Security)

Tables with user-owned data (`cart_items`, `bookmarks`, `orders`, `order_items`) have RLS enabled. Policies use `auth.uid()` to scope reads and writes to the current user. The service role client bypasses RLS entirely — used in webhooks and admin operations. The user client (built from the Bearer token) is subject to RLS — used for all authenticated user routes.

### CORS

Next.js handles preflight `OPTIONS` requests automatically when using the App Router. No manual CORS headers needed for same-origin and standard cross-origin requests.

