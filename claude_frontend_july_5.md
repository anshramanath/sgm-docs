# Dev Log — Sunglass Frontend (July 5, 2026)

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
- `POST /api/public/views` accepts `{ brandSlug, categoryId? | productSlug? }`
- Backend uses two Postgres RPCs (`increment_category_view`, `increment_product_view`) — atomic `UPDATE ... SET view_count = view_count + 1`, no read-then-write, no race condition
- `trackView()` in `api.ts` fires and forgets — no await at call site
- Called from `CategoryPage` (once per visit, not on filter change) and `ProductPage`
- `categories.view_count` defaults to `null` — only leaf nodes get tracked; `products.view_count` defaults to `0` — every product is a leaf

### Open Graph
- `og:title`, `og:description`, `og:image` set per brand in `generateMetadata`
- Image URL is `brand.url + brand.hero` — must be absolute since OG scrapers run server-side with no concept of your domain
- Platforms cache OG data aggressively — add a query param (`?v=2`) to bust the cache for testing

### Announcement Bar
- Messages moved from a hardcoded array in the component to `announcements: string[]` per brand in `brand.ts` — ready to be customised per brand

### Price / Sale Display Logic
- `formatPrice(cents)` — divides by 100, formats to 2 decimal places
- **Simple product** (has `sku`, no variations) — if `sale === true`, show regular price struck through + sale price in brand color
- **Variable product** (no `sku`, has variations) — `sale === true` only means at least one variant is on sale; show price range without strikethrough since other variants may be full price
- `ProductCard` detects simple vs variable via `variations.length === 0` (no `sku` needed at list level since there's no add-to-cart action)

---

## Bugs Fixed

### iOS Auto-Zoom on Sign-In Inputs
- **Cause:** iOS Safari auto-zooms any input with `font-size < 16px` on focus, leaving the page zoomed after keyboard dismisses
- **Fix:** Changed all three auth inputs from `text-[15px]` to `text-base` (16px)

### Horizontal Scroll on Checkout (Long Product Name)
- **Cause:** Left grid column `<div>` had no `min-w-0`. CSS Grid `fr` tracks default minimum to `min-content` — a long unwrapped name expanded the column beyond the viewport regardless of inner `truncate`
- **Fix:** `min-w-0` on the left grid column div

### Item Name Overflow in Bag Panel and Checkout
- **Pattern:** Inside a flex row, a text element needs both `min-w-0` (to shrink below content width) and `truncate`. Without `min-w-0`, `min-width: auto` = full text width and the element refuses to shrink
- **Fix:** `flex-1 min-w-0 truncate` on the name link, `flex-1 min-w-0` on the outer content div

### Cart Not Clearing on Order Success
- **Cause:** React runs child `useEffect`s before parent `useEffect`s. Success page's `clear()` fired before `CartProvider` had loaded from localStorage — cleared an already-empty Map, then provider loaded the real cart on top
- **Fix:** `clear()` fires right before the 5-second redirect, by which point the provider has finished loading

---

## Concepts

### `flex-1 min-w-0` Pattern
For text truncation inside flex rows. Without `min-w-0`, `min-width: auto` resolves to the full content width, so the element never shrinks below its text even with `truncate`. `min-w-0` drops the floor to 0.

### CSS Grid `min-content` Floor
`fr` tracks have a minimum of `min-content` by default. Non-wrapping content sets that floor regardless of the `fr` allocation. Fix: `min-w-0` on the grid item itself, not just elements inside it.

### React Effect Order
React commits `useEffect`s bottom-up — child effects run before parent effects. Don't rely on a parent context being hydrated at child mount time if the parent initialises asynchronously.

### Open Graph Absolute URLs
OG scrapers run on platform servers — no concept of your domain. All `og:image` values must be fully absolute URLs. Relative paths silently fail.

### localStorage Same-Origin Policy
`localStorage` is scoped strictly to origin (protocol + domain + port). Separate domains cannot share it regardless of knowing the key. Subdomains don't share it either, unlike cookies which can be scoped to a parent domain.

### iOS Input Zoom
Any `<input>` with `font-size < 16px` triggers auto-zoom on iOS Safari on focus. Zoom persists after keyboard dismisses. Fix: `font-size: 16px` or larger on all inputs.

### Atomic DB Increments
`UPDATE table SET col = col + 1` is a single atomic Postgres operation — no race condition. The race only exists when you read in application code first then write back. Supabase's query builder can't express `col + 1` as an expression, so an RPC is needed to keep it atomic without a two-step.

### POST vs GET for Side Effects
GET is for fetching and caching. POST is for actions with side effects. View tracking is an action — POST is semantically correct and avoids `revalidate` caching that applies to GET fetches.

### Order Number Display
Last 8 chars of the UUID uppercased: `order.id.slice(-8).toUpperCase()` → `#AB38DF09`. Unique enough for display, no extra column needed.

### Brand Scoping in localStorage
Keys are prefixed with `brandSlug` (e.g. `bikershades:cart`). Redundant if each brand is on its own domain (already isolated by origin), but correct if ever run on the same domain or localhost.
