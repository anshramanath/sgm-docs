# Dev Log (July 8, 2026)

## Features Built

### Multi-Brand Architecture
- Single Next.js codebase deployed under multiple domain names, each with its own `NEXT_PUBLIC_BRAND_SLUG` env var
- `src/lib/brand.ts` holds per-brand config: name, logo, colors, hero image, favicon, URL, description, announcement bar messages
- `getBrand()` reads `BRAND_SLUG` at runtime to return the right config — no conditionals scattered in components

### Announcement Bar
- Rotating text banner at the top of the shop layout
- Messages defined per-brand in `brand.ts` under `announcements: string[]`
- Two `<Sequence>` components rendered back-to-back and CSS-animated for a seamless infinite scroll
- Typed `messages: readonly string[]` because brand config uses `as const`, making arrays readonly

### Auth
- Supabase Auth via `@/lib/auth` server actions: `signIn`, `signUp`, `signOut`, `getUser`, `requireUser`
- `requireUser()` redirects to `/sign-in` if no session
- Sign-in page has a server-side auth guard — redirects to `/` if already logged in
- User metadata field is `name`, not `display_name`

### localStorage & Cross-Brand Isolation
- localStorage is scoped to `protocol + domain + port` — different domains can never share it, even if they know each other's keys
- Each brand deployed under its own domain is fully isolated by default
- Brand-prefixing localStorage keys is redundant when each brand has its own domain

### Cart
- Cart lives in a React context (`CartProvider`) wrapping the shop layout
- `clear()` on the order success page must run after the 5-second countdown, not on mount — child effects fire before parent effects, so calling `clear()` on mount races with `CartProvider`'s own load effect

### Checkout
- CSS Grid `fr` tracks have a `min-content` floor — a long product name in a grid column overflows regardless of `truncate` on the inner element
- Fix: `min-w-0` on the grid item div overrides the implicit `min-width: min-content`
- Same `flex-1 min-w-0` pattern applies to flex children that need to truncate text

### Open Graph / Link Previews
- Open Graph meta tags in `src/app/layout.tsx` via Next.js `generateMetadata`
- OG image URL must be absolute — scrapers run server-side and can't resolve relative paths
- `brand.url + brand.hero` concatenated to form the absolute image URL

### View Tracking
- `view_count` column on `categories` (default `null` — only leaf nodes get counts) and `products` (default `0`)
- Atomic increment via Postgres RPC — Supabase SDK can't express `col = col + 1`, so an RPC is required to avoid race conditions
- `trackView({ categoryId?, productSlug? })` in `src/lib/api.ts` — fires a POST to `/api/public/views`, not awaited (fire-and-forget)
- Called in `CategoryPage` and `ProductPage` directly, not inside `getProducts`/`getItem` helpers — navbar calls those helpers for featured products and would generate false counts
- `trackView` called in `CategoryPage`, not `ProductSection`, to avoid double-counting on filter changes

### Account Page — Order History
- Status model: `processing | shipped | refunded` only — `delivered` and `partially_refunded` removed
- Partial refund is derived: `refundedCents > 0 && status !== "refunded"` → shows "Partially Refunded"; `status === "refunded"` → shows "Refunded"
- STATUS map drives both label and color — processing = #737373, shipped = brand color, refunded = black
- Colors applied via inline `style={{ color, borderColor: color }}` — cleaner than a boolean toggle or Tailwind arbitrary values
- Carrier + tracking number shown as `{carrier} · {tracking}` (carrier grey, tracking dark) — only rendered when both fields are present
- Refunded label and amount both rendered in brand color

### Route Group Organization
- `(shop)/(footer)/` route group created to hold static info pages: about, contact, faq, privacy, returns, shipping, terms
- Route groups use `()` syntax — the segment is invisible to the router, so URLs are unchanged (`/about` not `/footer/about`)
- Pages inside `(shop)/(footer)` inherit the shop layout (navbar, footer) automatically — no extra layout file needed
- No imports, links, or paths needed updating — route groups are purely a filesystem organization tool

---

## Concepts

**CSS Grid min-content floor** — `fr` units won't shrink a column below `min-content`. Add `min-w-0` to the grid item to override.

**React effect order** — child `useEffect` fires before parent `useEffect`. Don't rely on a parent context being fully initialized when a child mounts.

**localStorage same-origin policy** — scoped to `protocol + domain + port`. Cross-domain isolation is enforced by the browser, not key naming.

**Open Graph absolute URLs** — link scrapers are server-side bots and can't resolve relative paths. Always use full `https://` URLs for OG images.

**Atomic DB increments** — `UPDATE SET col = col + 1` is safe under concurrent writes. Use an RPC when the SDK can't express the operation directly.

**POST vs GET** — GET is cacheable and idempotent. POST signals a side effect. View tracking, checkout, and mutations use POST.

**Fire-and-forget async** — don't `await` tracking calls in server components; the response doesn't affect the render and blocking adds latency.

**Derived state over extra DB columns** — partial refund is `refundedCents > 0 && status !== "refunded"`. No separate status value needed.

**Next.js route groups** — `(groupName)` folders are stripped from the URL. Used to share a layout across routes without polluting the path. Nesting route groups inherits parent layouts automatically.

**iOS input auto-zoom** — any `font-size < 16px` on an input triggers auto-zoom on focus in iOS Safari. Keep form inputs at `text-base` (16px) minimum.
