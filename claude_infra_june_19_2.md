# Sunglass Frontend — Notes (June 19, 2026)

## Stack

- **Next.js 16** (App Router, React Server Components, Suspense streaming)
- **Tailwind CSS v4** (arbitrary values, `ring-offset`, CSS variables in `globals.css`)
- **Supabase** (SSR auth via `@supabase/ssr`)
- **Stripe** (hosted checkout, redirect flow)
- **TypeScript** throughout

---

## Route Groups

| Group | Path | Layout |
|-------|------|--------|
| `(shop)` | `/`, `/category/[...path]`, `/product/[slug]`, `/sale` | Full nav + announcement bar + footer |
| `(bare)` | `/account`, `/checkout`, `/order/success`, `/order/failure`, `/signin` | Minimal header only |
| Root | `/not-found` | No layout chrome |

---

## Product Model

### Simple product
- `product.sku` is set, `product.variations` is `[]`
- `product.attributes` may list display attributes but no variation options

### Variable product
- `product.sku` is null, `product.variations` has entries
- Each variation has `sku`, `stock`, `attribute[]`, `images[]`, price fields
- `product.attributes` is the canonical pool of all attribute options

### Attribute option shape (both product-level and variation-level)
```ts
{ name: string; option: string; slug: string; value?: string }
// value is hex color — only present on color attributes
```

---

## Variation Selector Logic (ProductDetail)

### Attribute ordering
Color is always put first, regardless of API order:
```ts
const rawAttrNames = getAttrNames(product.variations);
const attrNames = rawAttrNames.includes("color")
  ? ["color", ...rawAttrNames.filter(n => n !== "color")]
  : rawAttrNames;
```

### Availability rules
- **Primary attr (index 0 / color)**: always fully available — `getAvailableOptions` is called with `{}` so nothing restricts it
- **Secondary attrs**: filtered by what variations exist given the other current selections
- Rationale: if you switch products via a category link, secondary options may not exist for the preselected color. Color should never become unclickable.

### Color swatches vs text buttons
- If any option in an attr has `value` (hex), the whole attr renders as circles
- Circle: `w-7 h-7 rounded-full border border-grey-200` with `ring-[1.5px] ring-ink ring-offset-1` on selected
- `ring-offset-1` creates a white gap between swatch and ring, making it visible on any background color

### URL slug resolution
URL carries `?color=gloss-black` (slug). On load, `defaultSelections` resolves it:
```ts
const bySlug = product.variations.flatMap(v => v.attribute)
  .find(a => a.name === n && a.slug === initialSelections[n]);
return [n, bySlug?.option ?? initialSelections[n]]; // fallback: treat as option name
```

---

## ProductCard (Category / List Pages)

### Color swatches
- Sourced from `product.variations` (list endpoint returns a simplified `ListVariation[]`)
- `ListVariation` shape: `{ id, attribute: [{ name, option, slug, value? }], imageSrc: string | null }`
- API dedupes by color slug — one entry per unique color
- Shows up to 5 swatches, then `+ N` for overflow

### Swatch interactions
- **Hover**: previews the variation's image (`hoveredVar` state), falls back to product thumbnail if `imageSrc` is null
- **Click**: navigates to `/product/[slug]?color=[colorSlug]` — ProductDetail then resolves slug → option on load
- Main card links (image, name, price) navigate to default product URL (no color param)
- Bookmark button is independent — does not navigate

### Key: `v.id` (stable variation ID from API)

---

## Cart

### Identity key
`${productSlug}:${sku}` — composite because two products can share a SKU

### Persistence
1. **localStorage**: immediate, via `useEffect` on `items` change (only after `loaded = true`)
2. **DB** (`putCart`): debounced 800ms after items change

### Clear race condition
On redirect back from Stripe, `CartProvider` runs `sync()` which fetches localStorage + DB and merges. If `clear()` runs before sync completes, sync overwrites the clear (DB still has items). Fix: success page has a 5-second countdown — `clear()` is only called after the timer expires, by which point sync has finished.

### `clear()` implementation
```ts
const clear = useCallback(() => setItems([]), []);
// localStorage auto-clears via persist useEffect when items → []
```

---

## Bookmarks

### Identity key
`productSlug` only — one bookmark per product regardless of variation

### Panel
- Opens as a Sheet (right drawer)
- Each item: image + name + Remove + View link
- View link mirrors the cart incrementer position (bottom-left of info column)

---

## Account Page

### Shipping address
Derived from the most recent order that has a `shippingAddress`:
```ts
const latestAddress = orders.find(o => o.shippingAddress)?.shippingAddress ?? null;
```
Displayed in the right column of Account Details.

### Layout
- Left col: Email, Name (if set), Member Since
- Right col: Shipping Address (if available, from latest order)
- Order footer: Total + shipping address inline (one-liner format)

### Order item address in footer
```
{name} · {line1}, {city}, {state} {postalCode}
```

---

## Checkout Flow

1. User reviews cart at `/checkout`
2. Click "Proceed to Payment" → `createCheckoutSession` → returns Stripe URL
3. `window.location.href = url` → redirect to Stripe hosted checkout
4. Success → `/order/success` (5s countdown → `clear()` → redirect home)
5. Failure → `/order/failure` (Try Again / Back to Shopping)

### Stripe success URL
```
${window.location.origin}/order/success
```
No `{CHECKOUT_SESSION_ID}` appended — we don't use it.

### Spinner during redirect
```tsx
<div className="w-9 h-9 border-2 border-grey-200 border-t-ink rounded-full animate-spin will-change-transform" />
```
`will-change-transform` moves the element to the GPU compositor thread so the CSS animation keeps running even when the main thread is busy processing the navigation redirect.

---

## Streaming Architecture (Product Page)

```tsx
export default async function ProductPage({ params, searchParams }) {
  const [{ slug }, { path, ...attrParams }] = await Promise.all([params, searchParams]);

  return (
    <>
      {/* Instant — from URL params, no fetch */}
      <Breadcrumb slug={slug} path={path} />

      <Suspense fallback={<DetailSkeleton />}>
        <Detail slug={slug} attrParams={attrParams} />   {/* fetches product */}
      </Suspense>

      <Suspense fallback={<RelatedSkeleton />}>
        <Related slug={slug} path={path} />              {/* fetches related */}
      </Suspense>
    </>
  );
}
```

### Breadcrumbs
Built from URL path segments — no fetch needed. `humanize(slug)` converts `gloss-black` → `Gloss Black` for display.

---

## 404 Handling

- **Root** `app/not-found.tsx`: bare page (no nav/footer), content pushed above center with `mb-[20vh]`
- **Shop 404s**: `notFound()` thrown from `(shop)` routes renders inside the shop layout (Next.js behaviour). No separate `(shop)/not-found.tsx` — accepted as-is.

---

## CSS / Design System

### Color tokens (`globals.css`)
```
--color-ink: #000000
--color-paper: #ffffff
--color-sale: #2EA3DC   (brand accent, also used for sale price)
--color-grey-50 → grey-950
```

### Custom animations
- `announcement-marquee`: horizontal scroll for the announcement bar
- `badge-pop`: scale pop for cart/bookmark count badge

### Key Tailwind patterns
- `ring-[1.5px] ring-ink ring-offset-1` — color swatch selection ring (offset creates white gap)
- `transition-[grid-template-rows]` — smooth accordion expand/collapse
- `mix-blend-multiply` — removes white backgrounds from product images on grey backgrounds
- `group` / `group-hover:scale-[1.04]` — card image zoom on hover

---

## Key Learnings

### `will-change-transform`
Forces element onto the GPU compositor thread. Keeps CSS animations (like `animate-spin`) running during heavy main-thread work such as a `window.location.href` navigation.

### React key stability
Use stable IDs (not slugs) as React keys. Slugs can change (SEO rewrites, renames), causing React to destroy and recreate DOM nodes unnecessarily. Doesn't break the UI — just wastes work.

### `ring-offset` for color swatches
`border` sits flush on the element edge and disappears against matching colors. `ring-offset-N` uses `box-shadow` to add a white gap between the element and the ring, making the selection indicator visible on any swatch color.

### Cart sync race condition
On fresh page load after Stripe redirect, `CartProvider.sync()` is async. It fetches DB (which still has cart items) and calls `setItems(merge(...))`. If `clear()` runs before sync resolves, sync overwrites the clear. Fix: delay `clear()` by enough time for sync to complete (5s countdown on success page).

### `notFound()` in route groups
Calling `notFound()` from a page inside `(shop)` renders the not-found UI inside the `(shop)` layout (nav, footer included). Only truly unmatched URLs hit the root `app/not-found.tsx` without any layout. To get a bare 404 from within a layout group, redirect to a route outside the group.

### Attribute availability — primary vs secondary
When two attributes exist (e.g. color + lens type), the primary (color) should always show all options. Only secondary options should be filtered based on what combinations exist. Otherwise switching products can leave the preselected color greyed out and unclickable.

### `product.attributes` vs variation attributes
`product.attributes` is the canonical pool of all possible options. `variation.attribute` is the specific combination for one SKU. For displaying all available options, use `product.attributes`. For resolving which variation is selected, use `product.variations`.
