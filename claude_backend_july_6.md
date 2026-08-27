# Admin Dashboard (July 6, 2026)

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
  products.ts       getProducts, getCategoryOptions

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
AdminProduct      — { id, name, slug, sale, featured, variable, minPriceCents, ... }
CategoryOption    — { id, name }
```

---

## Pages

### Overview
- Stats grid: catalogue stats (products, on sale, featured, categories) + order stats (active, completed, revenue, refunded)
- Revenue is brand-colored, active orders blue, completed orders green
- Recent products table uses `product_categories(categories!inner(name))` — outer join keeps orphaned products, inner ensures category name is never null
- Type column: `p.sku ? "Simple" : "Variable"` — top-level SKU means simple product

### Orders
- Status filter tabs: All / Processing / Shipped / Delivered / Refunded
- Expandable rows: click to reveal shipping address + line items
- When no orders: show all filter tabs with 0 counts, no rows

### Categories
- Flat array with `depth` (for CSS padding) and `ancestorIds` (for collapse visibility)
- CSS Grid requires flat siblings — nested DOM breaks column alignment
- `nodeMap: Record<string, string[]>` maps parent id → child ids only
- `rowMap` maps id → raw row for O(1) lookup during visit
- Collapse: `ancestorIds.every(id => !collapsed[id])` — hidden if any ancestor is collapsed
- View count: leaf nodes show actual count (brand color), parent nodes show "—"

### Analytics
- Two-stage fetch: summary first (totalViews, unitsSold), then three ranking functions in parallel
- `totalViews = productViews + categoryViews` — every page click counts
- Bar fill = item's count / total (share of total, not share of leader)
- Sales only count non-refunded orders: `.in("status", ["processing", "shipped", "delivered"])`
- Sales key = `${product_slug}:${sku}` — composite key handles multiple variants per product
- `order_items` has `product_slug`, `name`, `sku` (all non-nullable) — no separate products join needed
- `categories.view_count` is nullable (`??` needed); `products.view_count` is always an int

### Products
- Server component fetches initial page, defines `loadMore` as an inline server action closing over `brandSlug` — no separate `actions.ts` file
- Client component (`ProductsList`) receives `loadMore` as a prop; closure means it doesn't need `brandSlug`
- `initialSearch` prop = committed URL value; `search` state = what's typed in the input. They diverge while typing before Submit
- `applyCategory` preserves `initialSearch` (not the draft) — switching category doesn't submit a half-typed search
- `applySearch` pushes new URL → server re-renders with new `initialProducts`
- Category dropdown uses a fixed `position: fixed; inset: 0` backdrop div at `zIndex: 9` behind the panel at `zIndex: 10` — prevents page bleed-through and handles click-outside
- Selected category: accent background + white text in list, accent text in trigger label
- `variable = !p.sku` — no top-level SKU means the product has variations; matches overview logic
- Price: `!p.variable && p.sale` → strikethrough + sale price. Otherwise → price range
- Featured: accent color when true

#### SKU search
Simple products have a top-level `sku`. Variable products have no `sku` — their SKUs live on the `variations` table. So searching by SKU requires querying both:
1. `products` for `name.ilike` or `sku.ilike`
2. `variations` for `sku.ilike` → returns `product_id`

Both run in parallel, IDs are unioned via `Set`, then intersected with category filter IDs if active.

#### Pagination
Fetch `PAGE_SIZE + 1` rows (`.range(offset, offset + PAGE_SIZE)` — both ends inclusive, so 11 rows for PAGE_SIZE=10). If `data.length > PAGE_SIZE` → `hasMore = true`. Slice to `PAGE_SIZE` for display. Avoids a `COUNT(*)` query which scans all matching rows.

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
.range(offset, offset + PAGE_SIZE)  // inclusive both ends → PAGE_SIZE + 1 rows
const hasMore = data.length > PAGE_SIZE;
data.slice(0, PAGE_SIZE)
```

### Join conventions
- `relation(col)` — left join, relation array may be empty for orphaned rows
- `relation!inner(col)` — inner join, filters out rows with no match; use on nested joins to guarantee non-null, not on outer joins where you want all parent rows

### Error handling
```ts
if (error) throw new Error("Failed to fetch X");
```
Plain string, no interpolated `error.message`. `data` is guaranteed non-null after this check — no `?? []` needed.

### Casting Supabase results
When TypeScript's inferred type conflicts with the cast:
```ts
p.product_categories as unknown as { categories: { name: string } }[]
```

---

## Inline server actions

Server actions can be defined inside server components and passed as props to client components. The action closes over server-side values so the client never needs them:

```ts
// page.tsx (server component)
async function loadMore(search: string, categoryId: string, offset: number) {
  "use server";
  return getProducts(brandSlug, { search: search || undefined, categoryId: categoryId || undefined, offset });
}
```

Replaces a separate `actions.ts` file when the action is a thin wrapper.

---

## DB notes

- `order_items`: has `product_slug`, `name`, `sku` (non-nullable) — no join to products needed for analytics
- `products`: `featured` (bool), `sale` (bool), `sku` nullable (null = variable product), `view_count` (int)
- `categories`: `view_count` nullable
- `variations`: `product_id` FK to products, `sku` non-nullable — each variant has its own SKU
- `product_categories`: join table; empty array for orphaned products (no categories)

---

## Sidebar

Displays `user.user_metadata.name` (set via `supabase.auth.admin.updateUserById`) with email underneath. Falls back to email if no name set. Names are stored in `raw_user_meta_data` and can be set from the Supabase dashboard or via the admin API.
