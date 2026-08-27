# Admin Dashboard (July 4, 2026)

Multi-brand admin for BikerShades, proSPORT Sunglasses, and Sunglass Monster.

---

## What was built

### Auth (`src/lib/auth.ts`)

Central auth hub — all auth logic lives here, nothing scattered.

- `getUser()` — private, verifies JWT via `supabase.auth.getUser()`
- `requireAdmin()` — calls getUser + checks `admins` table, redirects to `/` on failure. Called at the top of every admin query function.
- `signIn(email, password)` — signs in, verifies admin row exists, redirects to `/admin/[firstBrandSlug]`
- `signOut()` — signs out

Two Supabase clients in scope → name them `authSupabase` / `adminSupabase`. One client → just `supabase`.

### Supabase clients

| Client | File | Key | Used for |
|---|---|---|---|
| `createAuthClient()` | `src/lib/supabase/server.ts` | `SUPABASE_ANON_KEY` | Cookie-based auth (JWT verify, sign in/out) |
| `createAdminClient()` | `src/lib/supabase/admin.ts` | `SUPABASE_SERVICE_ROLE_KEY` | All admin DB reads, bypasses RLS |

Public API routes use `src/lib/supabase/user.ts` (Bearer token) — untouched.

### Brand registry (`src/lib/brand.ts`)

Single source of truth for brand metadata. Replaced scattered hardcoded maps across files.

```ts
getBrandBySlug(slug)  // → brand or null
getAllBrands()         // → array of all brands
```

Each brand: `slug`, `name`, `accent` (hex), `logo` (public path).

### Shared types + utils

**`src/lib/types.ts`**
- `CategoryNode` — tree node with optional `children`
- `LeafEntry` — flattened leaf with `id`, `name`, `path`, `breadcrumbs`

**`src/lib/utils.ts`**
- `formatPrice(cents)` — formats cents as `$X.XX`
- `collectLeaves(tree)` — traverses a `CategoryNode[]` tree, returns a flat map of leaf nodes only (no parent categories)

### Admin layout (`src/app/admin/[brandSlug]/layout.tsx`)

Runs `requireAdmin()` on every page load. Renders `<Sidebar>` + `<main>`.

### Sidebar (`src/components/sidebar.tsx`)

Client component (`usePathname`, `useRouter`).

- Brand switcher — bordered pills. Active = accent fill + white text. Navigates to `/admin/[slug]` on click.
- Nav — Overview, Orders, Products, Categories, Analytics. Active = accent background. Overview uses exact path match; others use `startsWith`.
- Footer — user initials, email, "Admin" label, sign out button.
- Sticky (`sticky top-0 h-screen`) so it doesn't elongate with page content.
- Font: Hanken Grotesk via `--font-hanken` CSS var.
- Brand data sourced from `getAllBrands()`.

### Overview page (`src/app/admin/[brandSlug]/page.tsx`)

Server component. Runs three queries in parallel, renders results.

**Catalogue stats** — Products, On sale, Featured, Categories (leaf count only via `collectLeaves`)

**Order stats** — Active (processing), Completed (shipped + delivered), Revenue (sum of completed `total_cents`), Refunded (refunded + partially_refunded)

**Recent products table** — last 5 products with columns: image, Name, Type (Simple/Variable), Categories, Price, Sale, Featured

Price display logic:
- Has `sku` + on sale → strikethrough regular price + sale price in accent color
- Has `sku`, not on sale → regular price
- No `sku` (variable) → price range `$X – $Y`, or single price if min = max

### Admin query functions (`src/lib/admin/overview.ts`)

- Each function calls `await requireAdmin()` first — belt-and-suspenders gating even for reads
- Throws `new Error(...)` on DB failure — no result wrapper objects
- Uses `!inner` joins on `product_categories!inner(categories!inner(name))` so orphaned join rows are dropped at the DB level rather than filtered in JS

---

## Key decisions

**`requireAdmin()` on every query function** — even reads. Extra ~10ms of load time but means no unauthed data access even if a page somehow bypasses the layout guard.

**Throw errors, not result objects** — admin functions throw, server components catch via error boundaries. Simpler than a `good`/`bad` wrapper.

**`!inner` joins instead of `.filter(Boolean)`** — if a `product_categories` or `categories` row is missing, the DB drops the product entirely rather than returning nulls to filter in JS. Makes the data guarantee explicit.

**Leaf-only category count** — `collectLeaves()` traverses the tree and only counts nodes with no children, so parent categories aren't double-counted.

**Brand data in one place** — `brand.ts` is the single registry. `page.tsx`, `sidebar.tsx`, and `sign-in` all call `getAllBrands()` or `getBrandBySlug()` — no duplicate maps.

**Admin query functions separate from public API routes** — admin needs different fields (view counts, sale flags, etc.) and always has service-role access, so duplicating the query logic is intentional.

---

## File map

```
src/
  app/
    page.tsx                        ← sign-in page
    layout.tsx                      ← Hanken Grotesk, "Sunglass Server" title
    admin/[brandSlug]/
      layout.tsx                    ← requireAdmin + sidebar
      page.tsx                      ← overview
      orders/page.tsx               ← stub
      products/page.tsx             ← stub
      categories/page.tsx           ← stub
      analytics/page.tsx            ← stub
  components/
    sidebar.tsx
  lib/
    auth.ts                         ← getUser, requireAdmin, signIn, signOut
    brand.ts                        ← getBrandBySlug, getAllBrands
    types.ts                        ← CategoryNode, LeafEntry
    utils.ts                        ← formatPrice, collectLeaves
    admin/
      overview.ts                   ← getCatalogueStats, getOrderStats, getRecentProducts
    supabase/
      server.ts                     ← createAuthClient (cookie-based)
      admin.ts                      ← createAdminClient (service role)
      user.ts                       ← createUserClient (Bearer token, public API only)
public/
  bikershades/logo.jpg
  prosport-sunglasses/logo.jpg
  sunglass-monster/logo.jpg
```

---

## What's next

- Orders page + `src/lib/admin/orders.ts`
- Products page + `src/lib/admin/products.ts`
- Categories page + `src/lib/admin/categories.ts`
- Analytics page
