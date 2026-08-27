# What We Built & Learned (June 20, 2026)

## What We Built

A multi-tenant e-commerce API server built with Next.js App Router (API routes only). Each brand is a separate storefront sharing the same codebase and database.

### Endpoints

| Method | Path | Auth | Purpose |
|--------|------|------|---------|
| GET | `/api/public/brands` | No | List all brands |
| GET | `/api/public/categories` | No | Full category tree for a brand |
| GET | `/api/public/products` | No | Paginated products by category |
| GET | `/api/public/sale` | No | Paginated sale products |
| GET | `/api/public/item` | No | Full product detail |
| GET | `/api/public/search` | No | Product name search |
| POST | `/api/public/validate-cart` | No | Check cart items still exist |
| GET | `/api/user/cart` | Yes | Get user's cart |
| PUT | `/api/user/cart` | Yes | Replace user's cart |
| GET | `/api/user/bookmarks` | Yes | Get user's bookmarks |
| PUT | `/api/user/bookmarks` | Yes | Replace user's bookmarks |
| GET | `/api/user/orders` | Yes | Order history |
| POST | `/api/user/checkout` | Yes | Create Stripe checkout session |
| POST | `/api/webhooks/stripe` | Stripe sig | Handle payment completion |

### Database (Supabase / Postgres)

Tables: `brands`, `categories`, `products`, `variations`, `product_images`, `variation_images`, `product_description_images`, `description_images`, `product_categories`, `cart_items`, `bookmarks`, `orders`, `order_items`, `admins`

RPC functions: `increment_product_total_sales`, `increment_variation_total_sales`

Key schema decisions:
- Products belong to brands via `brand_slug` (not a FK join — simpler queries)
- Categories are self-referential (`parent_id`) for arbitrary depth trees
- `attributes` on products and `attribute` on variations are `jsonb` — flexible without extra tables
- `shipping_address` on orders is `jsonb` — mirrors Stripe's address object
- `stripe_session_id` unique constraint on orders makes the webhook idempotent
- Migration files: `001` is source of truth, numbered steps (`002`, `003`) record changes, live-apply files (`00N`) are run then deleted

---

## What We Learned

### Supabase Clients

Two different clients for two different purposes:

**Admin client** — uses the service role key, bypasses RLS entirely. Used for all public endpoints where you need to read data without a user.
```ts
import { createClient } from "@supabase/supabase-js";
createClient(url, serviceRoleKey, { auth: { autoRefreshToken: false, persistSession: false } });
```
`autoRefreshToken` and `persistSession` are disabled because server-side clients don't need session management — they're stateless per request.

**User client** — reads the `Authorization: Bearer <token>` header and creates a client scoped to that user. RLS policies on the DB then enforce that the user can only see their own rows.
```ts
import { createClient } from "@supabase/supabase-js";
createClient(url, anonKey, { global: { headers: { Authorization: `Bearer ${token}` } } });
```

`@supabase/ssr` is for cookie-based auth (same domain). Since the frontend and backend are on different domains, cookies aren't sent automatically — Bearer tokens are the right approach.

### Supabase Query Builder

Queries are built incrementally — each `.eq()`, `.gte()`, `.lte()` call returns a new query object and doesn't fire a request. The request only fires when you `await` it.

```ts
let q = supabase.from("products").select("...");
if (filter) q = q.eq("sale", true);       // still building
const { data } = await q.range(0, 19);    // fires here
```

`!inner` on a join filters the parent rows — only products that have a matching `product_categories` row are returned. Without `!inner` you get all products with the join result as null.

`gte` = `>=`, `lte` = `<=`.

### Pagination

`range(from, to)` is 0-indexed and inclusive. Page 1, size 20 → `range(0, 19)`.

`{ count: "exact" }` in `.select()` returns the total row count alongside data so you can compute `totalPages` and `hasNextPage` without a second query.

Load-more pattern only needs `hasNextPage` — `hasPreviousPage` is redundant because the frontend already knows the current page number.

### Stripe

**Checkout session** — pass `shipping_address_collection: { allowed_countries: ["US"] }` to make Stripe collect the address. No frontend form needed.

**Webhook** — in Stripe SDK v22, shipping details moved to `session.collected_information.shipping_details` (not `session.shipping_details`). The address is always populated in `checkout.session.completed` because Stripe enforces address collection before payment completes.

**Idempotency** — Stripe can deliver webhooks more than once. A unique constraint on `stripe_session_id` means the second insert fails silently, preventing duplicate orders.

### JSON & jsonb

- `jsonb` in Postgres stores structured data (objects, arrays) as binary. Faster to query than `json`.
- When data travels over the network it's serialized to a JSON string (bytes). The receiver parses it back into an object. `.json()` in Next.js does this serialization automatically.
- Adding a `NOT NULL` jsonb column to an existing table: use `default '{}'` temporarily, then `alter column ... drop default` immediately after — existing rows get `{}`, new rows must supply a value.

### Category Trees

Categories are stored flat with a `parent_id`. To build the tree: one pass to create a node map, one pass to attach children to parents. Sorting must be recursive (DFS) — a shallow sort only fixes the first level and misses grandchildren.

```ts
function sortTree(nodes: CategoryNode[]) {
  nodes.sort((a, b) => a.sortOrder - b.sortOrder);
  for (const node of nodes) {
    if (node.children) sortTree(node.children);
  }
}
```

DFS goes all the way down one branch before moving to the next. BFS would use a queue and process level by level — both work for sorting but DFS is simpler here.

### Color Variation Deduplication

A product can have many variations (e.g., Black Standard, Black Prescription, Red Standard, Red Prescription). For a product listing page you only want one swatch per color. Use a `Set` of color slugs inside a `reduce` to keep only the first occurrence:

```ts
const seen = new Set<string>();
variations.reduce((acc, v) => {
  const color = v.attribute.find((a) => a.name === "color");
  if (!color || seen.has(color.slug)) return acc;  // skip
  seen.add(color.slug);
  acc.push({ ... });
  return acc;
}, []);
```

`Array.find` returns the first element matching the condition (the object itself, not an index or boolean).

`reduce` is better than `map + filter` here because you can check `seen` and skip in one pass. `map` would produce nulls that you'd then need to filter out.

### Array Methods

| Method | Returns | Use when |
|--------|---------|----------|
| `map` | new array, same length | transform every element |
| `filter` | new array, shorter | remove elements |
| `find` | first matching element (or undefined) | locate one element |
| `reduce` | anything | build a new value from all elements |

### CORS

CORS only applies to browser-to-server requests. Server-to-server calls (Next.js API routes calling Supabase, frontend server components calling this API) don't trigger CORS checks — no headers needed.

### `ORDER BY` Performance

`ORDER BY` adds work — Postgres must sort the result set before returning it. Without an index on the sort column it's an in-memory sort. With an index it's faster but still not free. For unordered paginated data (load-more) removing the order clause is a valid optimization, with the tradeoff that row order is non-deterministic between pages.

### Data Shape Decisions

- Return only what the UI needs — don't over-fetch from the DB and don't over-return in the response.
- `imageSrc`/`imageName` at the product level: sort all images by `sort_order`, take index `[0]`. One image is enough for listing pages.
- Variation color swatches: `{ option, slug, value, imageSrc, imageName }` — flat, no wrapper array, no `name: "color"` field since the endpoint only returns color attributes.
- `value` (hex color) only exists on color attributes. All other attributes omit it.
- `sale` omitted from the sale endpoint response — it's always `true` by definition.
