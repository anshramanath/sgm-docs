# Frontend Notes (June 18, 2026)

## Stack

- **Next.js 16** App Router with TypeScript
- **Tailwind CSS** with custom token system (ink, paper, grey ladder, sale/brand accent via CSS vars)
- **Supabase SSR** for auth
- **Stripe** for checkout

---

## Route Groups

| Group | Layout | Purpose |
|-------|--------|---------|
| `(shop)` | AnnouncementBar + Navbar + Footer | All browsing pages |
| `(bare)` | None | Checkout, account, order pages, signin |

---

## Product Model

**Simple product** — `product.sku` is set, `product.variations` is empty. The product itself is the purchasable item.

**Variable product** — `product.sku` is null, `product.variations` is non-empty. Each variation has its own `sku`, `attribute[]`, images, and price. Only a resolved variation can be added to cart.

`ProductDetail` resolves which model applies via:
```ts
const hasVariations = product.variations.length > 0;
const sku = hasVariations ? (variation?.sku ?? null) : (product.sku ?? null);
```
Add to Bag is disabled when `sku` is null.

---

## Cart

**Key**: `${productSlug}:${sku}` — two products can share a SKU so slug+sku together is the unique identity.

**Persistence**: localStorage (immediate on state change) + debounced `putCart` to DB (800ms). On mount, local and DB are merged with DB winning conflicts.

**`clear()`**: only calls `setItems([])`. localStorage clears via the persist effect. The success page uses a 5-second countdown before calling `clear()` to ensure the DB sync has time to fire before navigation.

---

## Bookmarks

Keyed by `productSlug` only — one bookmark per product regardless of variation. No SKU involved. The bookmark API does not accept/return `sku` or `attribute`.

---

## Checkout Flow

1. `validateCart` (public endpoint) — checks stock, returns `Map<"slug:sku", boolean>`
2. Auth check via `getSession()` — redirects to `/signin` if not logged in
3. `createCheckoutSession` (authed) — returns Stripe URL
4. `window.location.href = url` — full navigation to Stripe
5. Success → `/order/success` (5s countdown → clear cart → redirect home)
6. Cancel → `/order/failure`

---

## Streaming (Product Page)

`page.tsx` resolves params immediately and renders:
- Breadcrumb — instant, built from URL path segments with `humanize(slug)`
- `<Suspense>` → `Detail` async component (fetches product)
- `<Suspense>` → `Related` async component (fetches category + products)

Both async components are defined inline in `page.tsx`.

---

## 404 Handling

- `app/not-found.tsx` — catches unmatched URLs, renders with root layout only (no shop chrome)
- `(shop)/not-found.tsx` — catches programmatic `notFound()` from shop pages, redirects to `/not-found` (bare page in `(bare)` group)

When `notFound()` is called from within a `(shop)` route, Next.js still wraps the output in the shop layout — hence the redirect approach.

---

## Key Learnings

**`notFound()` within a route group inherits the group's layout.** The root `app/not-found.tsx` only renders layout-free for truly unmatched URLs. For programmatic calls from within `(shop)`, you need a `(shop)/not-found.tsx` that redirects out.

**Cart clear race condition.** On the success page, `clear()` runs before `CartProvider`'s `sync()` completes (sync is async). By the time `sync()` finishes it overwrites the cleared state with DB items. Solution: delay `clear()` with a 5s timer so sync has finished.

**Sticky sidebar height.** A sticky aside taller than the viewport forces the page to always scroll. Fix: trim sidebar content so it fits within a typical viewport.

**CSS animations during navigation.** `will-change-transform` moves an element to the GPU compositor, keeping animations running even when the main thread is busy processing `window.location.href` navigation.

**`putCart` debounce + unmount.** If the user navigates away within 800ms of a cart change, React's cleanup cancels the debounce timeout and the DB never receives the update. Critical for the success page where the cart must be fully cleared before the next session.
