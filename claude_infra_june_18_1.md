# Sunglass Frontend — Dev Log (June 18, 2026)

## Project Overview

Multi-brand sunglass storefront built on Next.js 16 (App Router, Turbopack), Tailwind CSS, and Supabase SSR auth. The backend is a separate REST API; the frontend is purely a consumer.

---

## Architecture

### Route Groups
- `(shop)/` — public storefront pages (homepage, category, product, sale). Wrapped in Navbar + Footer.
- `(bare)/` — standalone pages with custom headers (signin, checkout, account). No shared nav.

### Key Files

| File | Role |
|------|------|
| `src/lib/brand.ts` | Single source of truth for brand config (name, logo, hero, colors, copy) |
| `src/lib/api.ts` | All API calls — `"use server"`, used as server actions from client components |
| `src/lib/auth.ts` | Supabase auth server actions: `signIn`, `signUp`, `signOut`, `getSession`, `requireUser` |
| `src/lib/categoryUtils.ts` | `collectLeaves()` — recursively walks the category tree, returns every leaf node with its full root-to-leaf path array |
| `src/lib/supabase/server.ts` | Creates a cookie-backed Supabase SSR client |
| `src/components/providers/AuthProvider.tsx` | Simple React context holding `loggedIn: boolean`. No Supabase listener — must be set manually via `setLoggedIn` |
| `src/components/providers/CartProvider.tsx` | Cart state with localStorage persistence + debounced DB sync via `putCart` |
| `src/components/providers/BookmarkProvider.tsx` | Bookmark state with localStorage persistence + debounced DB sync via `putBookmarks` |

---

## Auth Flow

### Sign In / Sign Up
Both forms (`SignInForm`, `SignUpForm`) follow the same pattern:
1. Call the server action (`signIn` / `signUp`)
2. Server action returns `null` on success or an error string
3. On success: `setLoggedIn(true)` then `router.push("/")`

`signUp` creates the account then immediately calls `signInWithPassword` so the user is signed in right away — no email confirmation step in the flow.

`AuthProvider` initializes `loggedIn: false` on every page load. It has no Supabase session listener. `loggedIn` is only ever `true` within a client session that explicitly set it via sign-in or sign-up. It doesn't survive page refreshes, but the Supabase session cookie does — protected pages use `requireUser()` (server-side) which reads from the cookie directly.

### `requireUser()`
Server-side only. Calls `supabase.auth.getUser()`, redirects to `/signin` if no user. Used on the account page.

### `getSession()`
Used by `authedFetch` in `api.ts` to get a bearer token for authenticated API calls. Reads the Supabase cookie server-side.

---

## Cart & Bookmark System

### Sync Strategy (both providers)
1. On mount (and when `loggedIn` changes): read localStorage → try DB → merge (DB wins on conflict) → set state
2. On every state change: write to localStorage immediately
3. On every state change (debounced 800ms): sync to DB via `putCart` / `putBookmarks`

### Cart Item Shape
```ts
type CartItem = {
  productSlug: string;
  sku: string | null;       // variation SKU — required for checkout and validate-cart
  attribute: { name: string; option: string }[];
  name: string;
  imageSrc: string;         // variation image if available, else product image
  priceCents: number;
  quantity: number;
}
```

`sku` is populated from `variation.sku` when adding from the product detail page. Falls back to `product.sku` for products with no variations.

### Image Resolution (ProductDetail)
```ts
const images = variation && variation.images.length > 0 ? variation.images : product.productImages;
```
Both cart and bookmarks use `images[0]` — the currently resolved image set for the selected variation.

---

## Checkout Flow

1. **Cart page** (`/checkout`): shows items, order summary, "Proceed to Payment" button
2. On mount: calls `validateCart` — dims and labels any items whose SKU no longer exists in the catalog
3. On "Proceed to Payment": filters to valid items with SKUs, calls `createCheckoutSession`, redirects to Stripe URL
4. `successUrl`: `/order/success?session_id={CHECKOUT_SESSION_ID}` (Stripe replaces the placeholder)
5. `cancelUrl`: `/checkout`

`/order/success` page does not exist yet.

---

## Homepage

| Section | Data source |
|---------|-------------|
| Hero CTAs | `collectLeaves` — "Shop Sunglasses" routes to the first leaf category |
| Top Categories | `collectLeaves` — first 5 leaves, each shows the first product image. 5th tile hidden on mobile |
| Best Sellers | First leaf category with ≥10 products |
| Editorial Split | Leaves at index 4 and 6. Second slot hidden on mobile (`hidden sm:flex`) |

---

## API Functions (`src/lib/api.ts`)

All functions are `"use server"`. Public endpoints use `apiFetch` (no auth). Authenticated endpoints use `authedFetch` (reads Supabase session cookie, attaches bearer token).

| Function | Endpoint |
|----------|----------|
| `getCategories` | `GET /api/public/categories` |
| `getProducts` | `GET /api/public/products` |
| `getSaleProducts` | `GET /api/public/sale` |
| `getItem` | `GET /api/public/item` |
| `searchProducts` | `GET /api/public/search` |
| `validateCart` | `POST /api/public/validate-cart` — returns `Map<sku, exists>` |
| `getCart` | `GET /api/user/cart` |
| `putCart` | `PUT /api/user/cart` |
| `getBookmarks` | `GET /api/user/bookmarks` |
| `putBookmarks` | `PUT /api/user/bookmarks` |
| `createCheckoutSession` | `POST /api/user/checkout` — returns Stripe URL |

---

## Debugging Notes

### Stale `.next` cache
Copying the project directory preserves Turbopack's cache with absolute paths baked in. Symptom: dev server compiles indefinitely. Fix: delete `.next/` before starting.

### `MallocNanoZone` flood
macOS env var `MallocNanoZone=0` causes Malloc log spam in Turbopack worker processes. Fix: `unset MallocNanoZone` before `npm run dev`.

### `turbopack: { root: __dirname }` in `next.config.ts`
Required to silence "could not determine workspace root" warning caused by two `package-lock.json` files on the machine (`~/package-lock.json` and the project root).

### `getSession()` and the signOut race
`getSession()` was calling `supabase.auth.signOut()` when `getUser()` returned null. This could clear a freshly-set session cookie if called in the same tick as a sign-up (before the cookie was applied to the request context). Removed the `signOut()` call — `getSession()` now returns `null` cleanly without side effects.

### Auth provider not updated on sign-up
Original `signUp` server action called `redirect("/")` server-side. The redirect happened before the client could call `setLoggedIn(true)`, so the auth context stayed stale. Fixed by returning `null` on success and handling navigation client-side (same pattern as `signIn`).

---

## Pending

- `/order/success` page — receives `session_id` from Stripe, shows order confirmation
- Cart page (standalone `/cart` route) is currently the same as `/checkout` — may want to split
- `AuthProvider` has no session listener — on page refresh `loggedIn` resets to `false`. Fine for now since nothing redirects based on it, but navbar sign-in/account state is server-rendered from the cookie anyway
