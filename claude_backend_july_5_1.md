# Admin Dashboard — Implementation Notes (July 5, 2026)

Built and maintained alongside `ADMIN_DESIGN_HANDOFF.md` (data spec) and `API.md` (public API). This document covers what was built, how, and why.

---

## Auth

Every admin query function calls `await requireAdmin()` at the top. This is belt-and-suspenders: the middleware already guards `/admin/*` routes, but query functions enforce it themselves so they're safe if ever called from elsewhere.

Two Supabase clients:
- `createAuthClient()` — cookie-based, used for auth checks (`requireAdmin`)
- `createAdminClient()` — service role key, bypasses RLS, used for all admin data queries

---

## Query Function Pattern

Each page section has its own exported async function in `src/lib/admin/`. Functions:
- Call `await requireAdmin()` first
- Throw errors (don't return result objects)
- Are called in parallel from the page via `Promise.all` where possible
- Are called sequentially when a later function depends on a result from an earlier one

Example of sequential dependency: analytics page fetches `getAnalyticsSummary` first (to get `totalViews` and `unitsSold`), then passes those into `getTopProductViews`, `getTopCategoryViews`, and `getTopProductSales` in parallel.

---

## Pages

### Overview — `src/app/admin/[brandSlug]/page.tsx`

Three parallel queries: `getCatalogueStats`, `getOrderStats`, `getRecentProducts`.

Stat tile colors: active orders = blue (`#2563eb`), completed orders = green (`#16a34a`), revenue = brand accent. Implemented via label-matching in the map rather than splitting the array.

`getRecentProducts` uses `.limit(5)` with no sort — products table has no `created_at` yet.

`getCatalogueStats` counts leaf categories only (`collectLeaves`). Uses `!inner` joins so products with no categories are excluded from the recent products list.

### Orders — `src/app/admin/[brandSlug]/orders/`

`getOrders` in `src/lib/admin/orders.ts`. Column is `stripe_payment_intent` (not `payment_intent`).

`orders-table.tsx` is a client component. Key decisions:
- `STATUS_LABEL` is the single source of truth — filter pills are derived from it via `Object.entries(STATUS_LABEL).map(...)`, so adding a new status only requires updating one constant
- Filter pills always show counts (show 0 when no orders), always show column headers
- Changing filter resets expanded row: `onClick={() => { setFilter(f.value); setExpanded(null); }}`
- Display ID: `#` + `id.slice(-8).toUpperCase()`
- Refunded amount shown only when `refundedCents > 0`

### Categories — `src/app/admin/[brandSlug]/categories/`

**Why flat array, not nested tree:** CSS Grid column alignment only works for direct children of the grid container. Nested DOM elements are in their own layout context. Since the table has five aligned columns (Name, Slug, Sort, Products, Views), all rows must be siblings — hence flat.

**How the flat array works:**
1. DB query returns rows sorted by `sort_order` (so children arrive in correct order)
2. `nodeMap: Record<string, string[]>` — maps each category ID to its children IDs (reverse of `parent_id`)
3. `rowMap: Record<string, Row>` — maps ID to full row data for O(1) lookup during traversal
4. DFS `visit()` walks root → children → grandchildren, building `FlatCategory[]` in display order
5. Each entry carries `depth` (for `paddingLeft` CSS only) and `ancestorIds` (for collapse filtering)

**Collapse logic:** `collapsed` state maps ID → boolean. A row is visible if `row.ancestorIds.every(id => !collapsed[id])`. Root nodes always have `ancestorIds: []` so `.every()` on an empty array is always true — roots can never be hidden.

**Why depth is only for UI:** depth is only used for `paddingLeft: row.depth * 28`. All logic uses `ancestorIds`.

**Sort badges:** root-level (depth=0) sort order rendered as a brand-colored rounded badge. Leaf names are lighter (`#525252`). Non-leaf Products and Views columns show `—`.

**View counts:** leaf nodes always show a number (0 if null). Non-leaves always show `—`. View count color is brand accent for leaves.

### Analytics — `src/app/admin/[brandSlug]/analytics/`

Four functions, fetched in two stages:

**Stage 1 (sequential):** `getAnalyticsSummary` — fetches products and categories for `totalViews`, filtered orders for `unitsSold`. Only `processing`, `shipped`, `delivered` orders count toward units sold (refunded excluded).

**Stage 2 (parallel):** `getTopProductViews(brandSlug, totalViews)`, `getTopCategoryViews(brandSlug, totalViews)`, `getTopProductSales(brandSlug, unitsSold)`.

**Bar percentage:** each bar is `item / total` (share of total), not `item / max`. This makes bars represent market share, not relative dominance within the leaderboard.

**Total views** = all product `view_count` + all category `view_count`. Every page click counts.

**`order_items` columns:** `product_slug`, `sku`, `name`, `image_src`, `price_cents`, `quantity`, `attribute`. No `product_id` — join back to products via `product_slug`.

Empty leaderboard: shows "No data yet." with no border separator.

---

## Key DB Decisions

**`!inner` joins** drop rows where the FK has no match at the DB level. Used in `getRecentProducts` on `product_categories!inner(categories!inner(name))` so products with no categories are excluded entirely rather than returning a null key.

**`stripe_payment_intent`** is the actual column name on `orders` (not `payment_intent`).

**`order_items.product_slug`** is how items link back to products. Populated at checkout from the Stripe line item product name (slugified).

**`view_count`** is nullable on both `products` and `categories`. Treat null as 0 for arithmetic; show `—` only where semantically appropriate (non-leaf nodes in categories).

---

## Types — `src/lib/types.ts`

Shared types used across admin and public:
- `FlatCategory` — flat row with `depth`, `ancestorIds`, `isLeaf`, `hasChildren`
- `Order` / `OrderItem` — admin orders
- `CategoryNode` — nested tree node (used by public API and nav)
- `LeafEntry` — leaf with breadcrumb path (used by `collectLeaves`)

---

## File Map

```
src/
  lib/
    admin/
      overview.ts    getCatalogueStats, getOrderStats, getRecentProducts
      orders.ts      getOrders
      categories.ts  getCategories → FlatCategory[]
      analytics.ts   getAnalyticsSummary, getTopProductViews, getTopCategoryViews, getTopProductSales
    types.ts         FlatCategory, Order, OrderItem, CategoryNode, LeafEntry
  app/
    admin/
      [brandSlug]/
        page.tsx                    Overview
        orders/
          page.tsx
          orders-table.tsx          Client component — filter pills, expandable rows
        categories/
          page.tsx
          categories-tree.tsx       Client component — collapsible grid
        analytics/
          page.tsx
```
