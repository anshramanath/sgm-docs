# Admin Dashboard (July 5, 2026)

Multi-brand sunglass e-commerce admin. Each brand has its own slug-routed section under `/admin/[brandSlug]/`.

---

## Auth

Every query function calls `requireAdmin()` at the top. This reads the session cookie via `createAuthClient()` and throws if the user is not an admin. All DB queries use `createAdminClient()` (service role key, bypasses RLS).

```ts
await requireAdmin();
const supabase = createAdminClient();
```

---

## File map

```
src/lib/admin/
  overview.ts       getCatalogueStats, getOrderStats, getRecentProducts
  categories.ts     getCategories → FlatCategory[]
  orders.ts         getOrders → Order[]
  analytics.ts      getAnalyticsSummary, getTopProductViews, getTopCategoryViews, getTopProductSales
  products.ts       getAdminProducts, getAdminCategoryOptions (re-exports types from @/lib/types)

src/app/admin/[brandSlug]/
  layout.tsx        sidebar with brand switcher + nav links
  page.tsx          overview
  orders/
    page.tsx        server component — passes orders + accent to OrdersTable
    orders-table.tsx  client component — status filter tabs, expandable rows
  categories/
    page.tsx        server component — passes flat array + accent to CategoriesTree
    categories-tree.tsx  client component — collapse/expand, grid rows
  analytics/
    page.tsx        server component — sequential fetch (summary first, then Promise.all rankings)
  products/
    page.tsx        server component — fetches products + categories, defines loadMore server action inline
    products-list.tsx  client component — search, category dropdown, paginated rows
```

---

## Types (`src/lib/types.ts`)

All shared types live here.

```ts
RankedRow         — { name, subtitle?, value, barPct }   used by analytics ranked lists
AnalyticsItemRow  — { product_slug, name, sku, quantity } used when aggregating order_items
Order / OrderItem — full order shape passed to orders-table
FlatCategory      — flat tree node with depth, ancestorIds, hasChildren, isLeaf
CategoryNode      — recursive tree node used for category tree building
LeafEntry         — leaf with path + breadcrumbs
AdminProduct      — { id, name, slug, sku, sale, featured, variable, minPriceCents, ... }
CategoryOption    — { id, name } for the products filter dropdown
```

`products.ts` re-exports `AdminProduct` and `CategoryOption` so callers don't need to know where they live:
```ts
export type { AdminProduct, CategoryOption };
```

---

## Pages

### Overview
- Stats grid: catalogue stats (products, on sale, featured, categories) + order stats (active, completed, revenue, refunded)
- Revenue is brand-colored, active orders blue, completed orders green
- Recent products table: same grid as products page
- Type column: `p.sku ? "Simple" : "Variable"` — top-level SKU means simple product

### Orders
- Status filter tabs: All / Processing / Shipped / Delivered / Refunded
- Expandable rows: click to reveal shipping address + line items
- When no orders: show all filter tabs with 0 counts, no rows
- `refunded_cents` is separate from `total_cents` — shown on partially refunded orders

### Categories
- Flat array with `depth` (for CSS padding) and `ancestorIds` (for collapse visibility)
- CSS Grid requires flat siblings — nested DOM breaks column alignment
- `nodeMap: Record<string, string[]>` maps parent id → child ids only (parentId removed as unused)
- `rowMap` maps id → raw row for O(1) lookup during visit
- Collapse: `ancestorIds.every(id => !collapsed[id])` — hidden if any ancestor is collapsed
- View count: leaf nodes show actual count (brand color), parent nodes show "—"
- `hasChildren` drives the expand/collapse arrow, `isLeaf` drives color

### Analytics
- Two-stage fetch: summary first (totalViews, unitsSold), then three ranking functions in parallel
- `totalViews = productViews + categoryViews` — every page click counts
- Bar fill = item's count / total (share of total, not share of leader)
- Sales only count non-refunded orders: `.in("status", ["processing", "shipped", "delivered"])`
- Sales key = `${product_slug}:${sku}` — composite key handles multiple variants per product
- `order_items` has `product_slug`, `name`, `sku` (all non-nullable) — no separate products join needed
- `categories.view_count` is nullable (`??` needed); `products.view_count` is always an int
- Empty ranked lists show "No data yet." with no border

### Products
- Server component fetches initial page, defines `loadMore` as an inline server action closing over `brandSlug` — no separate `actions.ts` file needed
- Client component (`ProductsList`) receives `loadMore` as a prop; the closure means it doesn't need `brandSlug`
- Pagination: `PAGE_SIZE = 20`, fetch `PAGE_SIZE + 1`, `hasMore = rows.length > PAGE_SIZE`, slice to `PAGE_SIZE`
- Category filter: separate subquery to `product_categories` gets product ids, then `.in("id", ids)` on main query — avoids join complexity
- Search: `.or("name.ilike.%term%,sku.ilike.%term%")`
- `variable = !p.sku` — matches overview page logic (top-level SKU = simple, no SKU = variable product with variations)
- Empty state: if `products.length === 0 && !q && !category` → "No products in this catalogue yet." (with border-top). Otherwise show the list with "No products match those filters." when filtered results are empty
- Category dropdown: brand-colored selected item (bg + white text), brand-colored trigger label text, fixed-position backdrop div behind panel at `zIndex: 9` (blocks page bleed-through and handles click-outside)
- State reset: `useEffect` on `searchParams.toString()` resets products/hasMore/searchDraft when URL changes; the `key` prop on `ProductsList` also forces remount on filter change — both work together

---

## Query patterns

### Parallel independent queries
```ts
const [a, b] = await Promise.all([queryA(), queryB()]);
```

### Sequential when later depends on earlier
```ts
const summary = await getAnalyticsSummary(brandSlug);
const [views, sales] = await Promise.all([
  getTopProductViews(brandSlug, summary.totalViews),
  getTopProductSales(brandSlug, summary.unitsSold),
]);
```

### Pagination (fetch N+1)
```ts
.range(offset, offset + PAGE_SIZE)  // fetches PAGE_SIZE + 1 rows
const hasMore = rows.length > PAGE_SIZE;
rows.slice(0, PAGE_SIZE)
```

### Casting Supabase results
When TypeScript's inferred type conflicts with the cast type, go through `unknown`:
```ts
p.product_categories as unknown as { categories: { name: string } | null }[]
```

---

## DB notes

- `order_items`: has `product_slug`, `name`, `sku` (no `product_id`) — no join to products needed for analytics
- `products`: `featured` (bool), `sale` (bool), `sku` nullable (null = variable product), `view_count` (int, non-nullable)
- `categories`: `view_count` nullable — use `?? 0` when summing
- `variations`: separate table with per-variant `sku`, `sale`, price fields
- `product_categories`: join table between products and categories

---

## Inline server actions

Server actions can be defined inside server components and passed as props to client components. The action closes over server-side values (like `brandSlug`) so the client never needs them:

```ts
// page.tsx (server component)
async function loadMore(search: string, categoryId: string, offset: number) {
  "use server";
  return getAdminProducts(brandSlug, { search: search || undefined, categoryId: categoryId || undefined, offset });
}

// passed as prop → client component calls it like a normal function
```

This replaces the pattern of a separate `actions.ts` file when the action is a thin wrapper.
