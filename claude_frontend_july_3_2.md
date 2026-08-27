# Dev Log — Sunglass Frontend (July 3, 2026)

## What Was Built

### Multi-Brand Architecture
- Single Next.js codebase deployed three times under different domains, each with its own brand config (`NEXT_PUBLIC_BRAND_SLUG` env var)
- `getBrand()` reads from a static `BRANDS` map keyed by slug — name, logo, hero, accent color, favicon, category images, editorial images, copy
- Brand accent color injected as a CSS custom property (`--color-brand`) on `<body>` so components can use `var(--color-brand)` without knowing the value

### Pages
- **Home** — hero, category grid, editorial strips, featured products
- **Category** — nested dynamic route (`/category/[...path]`), product grid with filters
- **Product** — variant selection, add to bag, save/unsave
- **Bag (slide panel)** — quantity controls, remove, subtotal, checkout link
- **Saved (slide panel)** — bookmarked products
- **Search (slide panel)** — debounced live search with featured fallback
- **Checkout** — review order, validate cart against backend, Stripe redirect
- **Order Success/Failure** — post-Stripe landing pages
- **Sign In / Sign Up** — Supabase auth, toggle between modes
- **Account** — signed-in state
- **Sale, FAQ, Privacy, Returns, Contact** — static/content pages

### Navbar
- Desktop: sticky header, hover dropdowns built recursively from category tree
- Mobile: hamburger opens a right-side Sheet panel with fully expanded category tree (recursive `NavLinks` component mirroring desktop `CategoryLinks`), sign-in link or account icon at top, Sale pill at bottom

### Providers
- `CartProvider` — localStorage + Supabase DB sync, merges on login, debounced writes
- `BookmarkProvider` — localStorage only
- `AuthProvider` — Supabase session state
- `Providers` — wraps all three

### Auth
- Supabase email/password sign in and sign up
- Server-side `getUser()` / `requireUser()` used in server components
- Sign-in page redirects to `/` if already authenticated

### Open Graph
- `og:title`, `og:description`, `og:image` set per brand using `brand.url + brand.hero` for an absolute image URL (required because OG scrapers run server-side and can't resolve relative paths)

---

## Bugs Fixed

### iOS Auto-Zoom on Sign-In Inputs
- **Cause:** iOS Safari auto-zooms any input with `font-size < 16px` on focus, leaving the page zoomed after the keyboard dismisses
- **Fix:** Changed all three auth inputs from `text-[15px]` to `text-base` (16px)

### Horizontal Scroll on Checkout (Long Product Name)
- **Cause:** The left column of the checkout grid (`<div>`) had no `min-w-0`. CSS Grid tracks using `fr` units default their minimum size to `min-content` — so a long unwrapped product name forced the column wider than the viewport regardless of inner `flex-1 min-w-0 truncate`
- **Fix:** Added `min-w-0` to the left grid column div. This drops the track's minimum to 0, letting the `fr` sizing win and the truncation inside work

### Item Name Overflow in Bag Panel and Checkout
- **Pattern:** Inside a flex row (`flex justify-between`), a text element needs both `min-w-0` (to allow shrinking below content width) and `truncate` (overflow hidden + nowrap + ellipsis). Without `min-w-0`, `min-width: auto` resolves to the full text width and the element refuses to shrink
- **Fix:** `flex-1 min-w-0 truncate` on the name link, `flex-1 min-w-0` on the outer content div

### Cart Not Clearing on Order Success
- **Cause:** React runs child `useEffect`s before parent `useEffect`s. The success page's `clear()` fired before `CartProvider`'s sync had loaded the cart from localStorage/DB — so it cleared an already-empty Map, then the provider loaded the real cart on top
- **Fix:** Moved `clear()` to fire right before the 5-second redirect, by which point the provider has finished loading

### Mobile Nav Horizontal Overflow
- **Cause:** `overflow-x-hidden` on `<body>` was clipping panel content rather than fixing the root cause
- **Removed:** the `overflow-x-hidden` and fixed the actual overflow sources instead

---

## CSS / Layout Concepts

### `flex-1 min-w-0` Pattern
The most important flex pattern for text truncation inside flex rows. Without `min-w-0`, a flex item's minimum width is its content width (`min-width: auto`), so it will never shrink below the full text width even with `truncate`. `min-w-0` overrides this to 0, making the element shrinkable.

### CSS Grid `min-content` Floor
`fr` units in CSS Grid don't work like pure fractions — each track has a minimum size of `min-content` by default. Any content that can't wrap (long words, `white-space: nowrap`) sets that floor. Fix: add `min-w-0` to the grid item, not just to elements inside it.

### Flex Effect Order (Parent vs Child)
React commits `useEffect`s bottom-up (children before parents). If a child component calls a state setter from a parent context on mount, the parent's own mount effect hasn't run yet — so the parent's state may not be initialized. Design around this: don't call context setters on mount if the context initializes asynchronously.

### Open Graph Absolute URLs
OG scrapers (iMessage, Slack, Twitter, etc.) fetch pages from their own servers — they have no concept of your domain. All `og:image` values must be full absolute URLs (`https://yourdomain.com/image.jpg`). Relative paths silently fail.

### localStorage Same-Origin Policy
`localStorage` is scoped to `origin` (protocol + domain + port). Separate domains cannot share it regardless of knowing the key. Subdomains don't share it either (unlike cookies, which can be scoped to a parent domain with `Domain=.example.com`).

### iOS Input Zoom
Any `<input>` with `font-size < 16px` triggers auto-zoom on iOS Safari when focused. The zoom persists after the keyboard dismisses, creating apparent horizontal scroll. The fix is always `font-size: 16px` or larger on inputs.
