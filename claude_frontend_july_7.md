# Dev Log (July 7, 2026)

## Features Built

### Multi-Brand Architecture
- Single Next.js codebase deployed under multiple domain names, each with its own `NEXT_PUBLIC_BRAND_SLUG` env var
- `src/lib/brand.ts` holds per-brand config: name, logo, colors, hero image, favicon, URL, description, announcement bar messages
- `getBrand()` reads `BRAND_SLUG` at runtime to return the right config — no conditionals scattered in components

### Announcement Bar
- Rotating text banner at the top of the shop layout
- Messages are defined per-brand in `brand.ts` under `announcements: string[]`
- Two `<Sequence>` components rendered back-to-back and CSS-animated to create a seamless infinite scroll effect
- Typed `messages: readonly string[]` because brand config uses `as const`, making arrays readonly

### Auth
- Supabase Auth via `@/lib/auth` server actions: `signIn`, `signUp`, `signOut`, `getUser`, `requireUser`
- `requireUser()` redirects to `/sign-in` if no session; used in account and other protected pages
- Sign-in page has an auth guard: fetches user server-side and redirects to `/` if already logged in
- `display_name` renamed to `name` in Supabase `user_metadata` — simpler and consistent with Supabase conventions

### localStorage Scoping
- localStorage is scoped to `protocol + domain + port` — different domains can never share it, even if they know each other's keys
- Each brand deployed under its own domain is fully isolated in localStorage by default
- Brand-prefixing keys in localStorage is redundant when each brand has its own domain, but harmless

### Cart
- Cart lives in a React context (`CartProvider`) wrapping the shop layout
- Cart state is synced to/from the backend on mount and on sign-in
- `clear()` must run after the 5-second countdown on the order success page, not on mount — child effects fire before parent effects, so calling `clear()` on mount races with `CartProvider`'s load

### Checkout
- CSS Grid `fr` tracks have a `min-content` floor by default — a long product name in a grid column will overflow the container regardless of `truncate` on the inner element
- Fix: add `min-w-0` to the grid item div to override the implicit `min-width: min-content`
- Same `flex-1 min-w-0` pattern applies to flex children that need to truncate text

### Open Graph / Link Previews
- Open Graph meta tags added in `src/app/layout.tsx` via Next.js `generateMetadata`
- Image URL must be absolute — scrapers run server-side and can't resolve relative paths
- `brand.url` (e.g. `https://example.com`) + `brand.hero` (e.g. `/hero.jpg`) concatenated to form the absolute URL
- `brand.url` and `brand.hero` added to each brand config in `brand.ts`

### View Tracking
- `view_count` column added to `categories` (default `null` — only leaf nodes receive counts) and `products` (default `0`)
- Race conditions: SQL `UPDATE SET col = col + 1` is atomic at the DB level; two simultaneous updates won't clobber each other
- Supabase JS SDK can't express `col = col + 1` — requires a Postgres RPC
- Two RPCs: `increment_category_view(p_id, p_brand_slug)` and `increment_product_view(p_slug, p_brand_slug)`
- `trackView({ categoryId?, productSlug? })` in `src/lib/api.ts` — fires a POST to `/api/public/views`, not awaited at call sites (fire-and-forget)
- POST vs GET: GET is for fetching/caching, POST for side effects — view tracking uses POST with a JSON body
- Call sites: `CategoryPage` (not `ProductSection`) to avoid double-counting on filter changes; `ProductPage` directly
- Not called inside `getProducts` or `getItem` helpers — navbar calls those for featured products and would generate false counts

### Account Page — Order History
- Orders fetched server-side via `getOrders()` and rendered statically
- Status model: `processing | shipped | refunded` only — `delivered` and `partially_refunded` removed
- Partial refund is derived state: `refundedCents > 0 && status !== "refunded"` → label shows "Partially Refunded"; full refund shows "Refunded"
- STATUS map drives both label and color: processing = #737373, shipped = brand color, refunded = black
- Colors applied via inline `style={{ color, borderColor: color }}` — avoids needing Tailwind arbitrary values or a boolean toggle
- Carrier + tracking number shown together as `{carrier} · {trackingNumber}`, carrier grey, tracking dark — only rendered when both are present
- Refunded line is brand color throughout (label + amount)

---

## Concepts

**CSS Grid min-content floor** — `fr` units won't shrink a column below its content's `min-content` size. Add `min-w-0` to override.

**React effect order** — child `useEffect` fires before parent `useEffect`. Don't rely on a parent context being fully initialized when a child mounts.

**localStorage same-origin policy** — scoped to `protocol + domain + port`. Cross-domain isolation is enforced by the browser, not by key naming.

**Open Graph absolute URLs** — link scrapers are server-side bots; they can't resolve `/relative/paths`. Always use full `https://` URLs for OG images.

**Atomic DB increments** — `UPDATE SET col = col + 1` is safe under concurrent writes. Use an RPC when the SDK can't express the operation.

**POST vs GET** — GET is cacheable and idempotent (fetching data). POST signals a side effect (view tracking, checkout). Never use GET for mutations.

**Fire-and-forget async** — don't `await` tracking calls in server components; the response doesn't affect the page render and blocking on it adds latency.

**Derived state over extra DB columns** — partial refund is `refundedCents > 0 && status !== "refunded"`. No separate status value needed.

**iOS input auto-zoom** — any `font-size < 16px` on an input triggers auto-zoom on focus in iOS Safari. Keep form inputs at `text-base` (16px) minimum.
