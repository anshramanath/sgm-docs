# Dev Log — Sunglass Frontend (July 4, 2026)

## What Was Built

### Multi-Brand Architecture
- Single Next.js codebase deployed three times under different domains, each driven by `NEXT_PUBLIC_BRAND_SLUG`
- `getBrand()` reads a static `BRANDS` map — name, logo, hero, favicon, accent color, category images, editorial images, announcement messages, hero copy
- Brand accent injected as `--color-brand` CSS custom property on `<body>`
- Each deployment has isolated localStorage and its own Supabase session cookie — users must sign in separately per brand (different domains, same-origin policy)

### Pages
- **Home** — hero, category grid, editorial strips, featured products
- **Category** — nested dynamic route (`/category/[...path]`), product grid, filter pills, load more
- **Product** — variant selection, add to bag, save/unsave
- **Bag (slide panel)** — quantity controls, remove, subtotal, checkout link
- **Saved (slide panel)** — bookmarked products
- **Search (slide panel)** — debounced live search with featured fallback
- **Checkout** — review order, validate cart against backend, Stripe redirect
- **Order Success/Failure** — post-Stripe landing pages, cart cleared before redirect
- **Sign In / Sign Up** — Supabase auth, redirects to `/` if already signed in
- **Account** — order history, order number displayed as last 8 chars of UUID uppercased (`#AB38DF09`)
- **Sale, FAQ, Privacy, Returns, Contact** — static/content pages

### Navbar
- Desktop: sticky header, hover dropdowns built recursively from category tree (`CategoryLinks`)
- Mobile: hamburger opens a right-side Sheet panel with fully expanded category tree (`NavLinks` — recursive, mirrors desktop), sign-in link or account icon at top, Sale pill at bottom

### Providers
- `CartProvider` — localStorage + Supabase DB sync, merges on login, debounced writes
- `BookmarkProvider` — localStorage only
- `AuthProvider` — Supabase session state

### View Tracking
- `POST /api/public/views` endpoint accepts `{ brandSlug, categoryId? | productSlug? }`
- Backend uses two Postgres RPCs (`increment_category_view`, `increment_product_view`) to do atomic `UPDATE ... SET view_count = view_count + 1` — no read-then-write, no race condition
- `trackView()` in `api.ts` calls the endpoint fire-and-forget (no await at call site)
- Called from `CategoryPage` (once per visit, not on filter change) and `ProductPage`
- `categories.view_count` defaults to `null` — only leaf nodes get tracked; `products.view_count` defaults to `0` — every product is a leaf

### Open Graph
- `og:title`, `og:description`, `og:image` set per brand in `generateMetadata`
- Image URL constructed as `brand.url + brand.hero` — must be absolute since OG scrapers fetch server-side with no concept of your domain

### Announcement Bar
- Messages moved from a hardcoded array in the component to `announcements: string[]` on each brand in `brand.ts` — ready to be customised per brand

---

## Bugs Fixed

### iOS Auto-Zoom on Sign-In Inputs
- **Cause:** iOS Safari auto-zooms any input with `font-size < 16px` on focus, leaving the page zoomed after keyboard dismisses
- **Fix:** Changed all three auth inputs from `text-[15px]` to `text-base` (16px)

### Horizontal Scroll on Checkout (Long Product Name)
- **Cause:** The left column `<div>` of the checkout grid had no `min-w-0`. CSS Grid `fr` tracks default their minimum to `min-content` — a long unwrapped product name expanded the column beyond the viewport regardless of inner `truncate`
- **Fix:** `min-w-0` on the left grid column div

### Item Name Overflow in Bag Panel and Checkout
- **Pattern:** Inside a flex row, a text element needs both `min-w-0` (shrink below content width) and `truncate`. Without `min-w-0`, `min-width: auto` equals the full text width and the element won't shrink
- **Fix:** `flex-1 min-w-0 truncate` on the name link, `flex-1 min-w-0` on the outer content div

### Cart Not Clearing on Order Success
- **Cause:** React runs child `useEffect`s before parent `useEffect`s. The success page's `clear()` fired before `CartProvider`'s sync had loaded from localStorage — clearing an already-empty Map, then the provider loaded the real cart on top
- **Fix:** `clear()` fires right before the 5-second redirect, by which point the provider has finished loading

---

## CSS / Layout Concepts

### `flex-1 min-w-0` Pattern
For text truncation inside flex rows. Without `min-w-0`, a flex item's `min-width: auto` resolves to its content width, so it never shrinks below full text width even with `truncate`. `min-w-0` overrides this to 0.

### CSS Grid `min-content` Floor
`fr` tracks have a minimum of `min-content` by default. Any non-wrapping content sets that floor regardless of the `fr` allocation. Fix: `min-w-0` on the grid item itself, not just elements inside it.

### React Effect Order (Parent vs Child)
React commits `useEffect`s bottom-up — child effects run before parent effects. If a child calls a context setter on mount while the parent's own mount effect is async and not yet complete, the parent's state may not be initialized yet. Design around this: don't rely on context being hydrated at child mount time.

### Open Graph Absolute URLs
OG scrapers run on platform servers — they have no concept of your domain. All `og:image` values must be fully absolute URLs. Relative paths silently fail.

### localStorage Same-Origin Policy
`localStorage` is scoped strictly to origin (protocol + domain + port). Separate domains cannot share it regardless of knowing the key. Subdomains don't share it either, unlike cookies which can be scoped to a parent domain.

### iOS Input Zoom
Any `<input>` with `font-size < 16px` triggers auto-zoom on iOS Safari when focused. The zoom persists after the keyboard dismisses. Fix: always `font-size: 16px` or larger on inputs.

### Atomic DB Increments
`UPDATE table SET col = col + 1` is a single atomic operation in Postgres — no read involved, no race condition. The race only exists when you read in application code first, then write back. Supabase's query builder can't express `col + 1` as an expression, so an RPC is needed to keep it atomic.

### POST vs GET for Side Effects
GET + query params is for fetching/caching. POST + body is for actions with side effects. View tracking is an action — POST is semantically correct and prevents caching (`revalidate` only applies to GET fetches).

### Order Number Display
Last 8 characters of the UUID, uppercased: `order.id.slice(-8).toUpperCase()` → `#AB38DF09`. Unique enough for display, no extra column needed.
