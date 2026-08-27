# Sunglass Frontend — Last 15 Commits: Everything Built & Learnt (June 27, 2026 - July 1, 2026)

Covers commits `44192ba` → `701908e`

---

## What Was Built

### `src/lib/utils.ts`
**Added `formatPrice(cents: number): string`**
- Centralised price formatting: `$${(cents / 100).toFixed(2)}`
- Previously every file had its own local `fmt` or `formatPrice` function with slightly different behaviour (some did `d % 1 === 0 ? $d : $d.toFixed(2)`, others always `.toFixed(2)`). Standardised to always two decimal places.
- Imported and used across: `account/page.tsx`, `checkout/page.tsx`, `ProductDetail.tsx`, `ProductCard.tsx`, `HeaderIcons.tsx`

---

### `src/lib/brand.ts`
**Added `shortName` and `url` fields to every brand. Added `getBrands()`.**
- `shortName` — compact label for UI (e.g. "SGM" for Sunglass Monster)
- `url` — deployment URL per brand (placeholder vercel URLs for now)
- `getBrands()` — returns all brands sorted so the current brand is first:
  ```ts
  .sort((a, b) => a.slug === current.slug ? -1 : b.slug === current.slug ? 1 : 0)
  ```
  Sort comparator: negative = put `a` first, positive = put `b` first, 0 = equal.

---

### `src/lib/types.ts`
**Tightened `ProductListItem`:**
- `imageSrc: string | null` → `string` (API always returns an image)
- `imageName: string | null` → `string`
- `variations?: ListVariation[]` → `variations: ListVariation[]` (API always returns an array)
- Removed all `?? []` fallbacks and null guards downstream as a result

---

### `src/lib/api.ts`
- Changed `SERVER_BASE_URL` → `NEXT_PUBLIC_SERVER_BASE_URL` (needs to be public for client components to read it)
- Changed relative import `./types` → absolute `@/lib/types`

---

### `src/components/layout/AnnouncementBar.tsx`
**Wired to `brand.ts` instead of hardcoded data.**
- Was: `const BRANDS = [{ label: "Monster", href: "#" }, ...]`
- Now: calls `getBrand()` and `getBrands()`, active brand determined by slug match
- Messages in the scrolling sequence changed from `<a>` links to `<span>` — messages don't navigate anywhere

---

### `src/components/layout/HeaderIcons.tsx`
**Major refactor — largest change in this set of commits.**

**`onClose` prop removed from all panels:**
- Was: `BagPanelContent({ onClose })`, `SavedPanelContent({ onClose })`, `SearchPanelContent({ featured, onClose })`
- Now: all three take no close-related props
- Sheet closure handled by `SheetClose asChild` wrapping navigation links — merges close behaviour onto the child element without adding a wrapper button

**`SheetClose asChild` pattern:**
```tsx
<SheetClose asChild>
  <Link href="/product/slug">...</Link>
</SheetClose>
```
Avoids nested button/anchor issues. When the link is clicked, the sheet closes automatically.

**`BagPanelContent` restructure:**
- Was: always-rendered scrollable div with conditional empty message inside
- Now: ternary at the top level — empty state vs `<>scroll div + footer</>`. This lets the footer be outside the scroll area cleanly.
- Key `attrParams` URL building removed `encodeURIComponent` — attribute names and slugs are safe without it

**`SavedPanelContent`:**
- Was: `items.length` for the count badge
- Now: `useBookmarkCount()` hook — more semantic, consistent with cart pattern

**`SearchPanelContent`:**
- Debounce converted from `.then().catch().finally()` chain to `async/await` with try/catch/finally inside `setTimeout`
- Cleanup: `return () => clearTimeout(timeout)` — cancels pending search when query changes before the 300ms fires
- Product links now use `SheetClose asChild`
- Added Sale + Best Seller badges to search result images
- Price logic updated (see price section below)

**Badge counts:**
- Was: `bookmarks.length` from `useBookmarkItems()` (allocates full array just for a count)
- Now: `useBookmarkCount()` reads `.size` directly from the Map

---

### `src/components/layout/Navbar.tsx`
- Dropdown trigger `<span>` removed `hover:underline` — words that open a dropdown shouldn't underline, only clickable leaf links should
- Leaf links in `CategoryLinks` gained `hover:underline underline-offset-[4px] decoration-1`
- Dropdown font size `text-[16px]` → `text-[15px]` to match top-level nav links
- `cat.children && cat.children.length > 0` → `cat.children` (children is only present if non-empty per API contract)

---

### `src/components/layout/NavMenu.tsx` — **Deleted**
Was an unused component using `@base-ui/react/navigation-menu`. Replaced by the custom `Navbar.tsx` implementation.

---

### `src/components/ui/*` — **Deleted (5 files)**
All unused shadcn-style components:
- `card.tsx`, `navigation-menu.tsx`, `separator.tsx`, `skeleton.tsx`, `tabs.tsx`

The `ui/` directory is now empty (or gone).

---

### `src/components/shared/Sheet.tsx`
Moved from `src/components/ui/sheet.tsx` → `src/components/shared/Sheet.tsx`. Renamed to PascalCase to match other shared components.

---

### `src/components/product/ProductCard.tsx`
**`categoryPath` prop removed — now derived internally:**
- Was: `ProductCard({ product, categoryPath?: string })`
- Now: `ProductCard({ product })` — calls `usePathname()` and slices `/category/` prefix if present
- This removed a prop that had to be threaded through `ProductGrid` → `ProductCard`

**Color swatch buttons → Links:**
- Was: `<button onClick={() => router.push(...)}>`
- Now: `<Link href={href}>` with `onMouseEnter` still setting hovered state
- Simpler, native navigation, no need for `useRouter`

**Overflow swatch count:** `<Link>+{overflow}</Link>` → `<span>+{overflow}</span>` — it's decorative, not navigable on its own

**`useIsBookmarked` signature change:** was `useIsBookmarked(product.slug)`, now `useIsBookmarked(product)` — takes the whole object

**Sale price logic fix:**
- Was: `product.salePriceCents ?` — presence of a sale price value was used to determine if on sale
- Now: `product.sale && !product.variations.length` — `sale` boolean is the source of truth; strikethrough only for simple (non-variable) products
- Variable products always show range or single price, never strikethrough (can't know which variation is on sale from the list view)

---

### `src/components/product/ProductGrid.tsx`
- `categoryPath` prop removed, just passes `product` to `ProductCard`

---

### `src/components/product/LoadMoreProducts.tsx`
**Moved and made generic.**
- Was: `src/app/(shop)/category/[...path]/LoadMoreProducts.tsx` with hardcoded `getProducts({ categoryId, filter })`
- Now: `src/components/product/LoadMoreProducts.tsx` with `fetchPage: (page: number) => Promise<{ products, hasNextPage }>` callback
- Both category and sale pages now use it via thin wrappers (`LoadMore.tsx` in each route) that close over their own fetch function

---

### `src/components/providers/BookmarkProvider.tsx`
- Added `useBookmarkCount()` — returns `items.size` directly, avoids spreading the Map
- `useIsBookmarked` changed from `(slug: string)` to `(item: { slug: string })` — consistent with how it's called
- localStorage key changed: `bookmarks:${brandSlug}` → `${brandSlug}:bookmarks` (namespace first)
- Debounce for `putBookmarks` now catches `Unauthorized` and calls `setLoggedIn(false)` instead of swallowing the error

---

### `src/components/providers/CartProvider.tsx`
- Added `useUpdateCartPrice(item, priceCents)` — mutates price in place in the cart Map, used by checkout validation
- localStorage key changed: `cart:${brandSlug}` → `${brandSlug}:cart`
- Same Unauthorized error handling pattern as BookmarkProvider

---

### `src/components/providers/Providers.tsx`
- Changed relative imports (`"./AuthProvider"`) to absolute (`"@/components/providers/AuthProvider"`)

---

### `src/app/(shop)/page.tsx`
**Fixed `leaves &&` rendering `0`:**
- Was: `{leaves && brand.categoryImages.map(...)}` and `{leaves && brand.editorial.map(...)}`
- Now: `{leaves.length > 0 && ...}`
- Root cause: empty array `[]` is truthy in JS, so `[] && expr` evaluates `expr`. When `leaves` is `[]`, `leaves.length` is `0`, and `0 &&` short-circuits to `0` which React renders as the character `0`. Fixed to `leaves.length > 0 &&`.

---

### `src/app/(shop)/product/[slug]/ProductDetail.tsx`
**Complete rewrite of selection and price logic.**

**`selections: Record<string, string>` (not `Record<string, string | null>`):**
- Was: all attrs initialised to their first variation's slug or `null`
- Now: starts empty or from URL params, only contains attrs the user has explicitly chosen
- Cleaner — absence from the map = not selected

**`availableByAttr` computed once per render:**
- Only computed for attrs NOT yet in selections
- For selected attrs, "available" doesn't apply — they're already chosen
- `getAvailableOptions(variations, attrName, otherSelections)` — filters variations matching all other selections, then returns the set of slugs for the target attr

**`select()` — four cases:**
1. Same attr, same slug → deselect (remove from Record with `const { [attrName]: _, ...rest } = selections`)
2. Same attr, different slug → reset all selections to just this one (switching forces reconsideration of everything)
3. Attr not yet selected, option unavailable → force-select anyway (reset to just this)
4. Attr not yet selected, option available → add to existing selections

**`isAvailable` short-circuit:**
```ts
const isAvailable = attrSelected ? true : available.has(o.slug);
```
When the attr is already selected, `available` is undefined (not in `availableByAttr`). Short-circuit prevents calling `.has` on undefined.

**Price display using `!sku` as the guard:**
- `!sku` = no variation resolved yet (variable product with no full selection, or simple product would always have sku)
- `!sku` → show range or single from product-level `minPriceCents`/`maxPriceCents`
- `onSale` → strikethrough (safe here: either it's a simple product, or a specific variation is selected)
- else → `regularPriceCents`

**Unavailable options — diagonal line instead of `disabled`:**
- Was: `disabled` attribute + `cursor-not-allowed` + `opacity-30`
- Now: clickable (still selects, resetting state), with an SVG diagonal line overlay
- Color circles: white SVG line `stroke="rgba(255,255,255,0.85)"`
- Size boxes: `currentColor` line (inherits the grey text colour)

**`handleBookmark` wrapper removed:**
- Was: `function handleBookmark() { const thumbnail = ...; if (!thumbnail) return; toggleBookmark(...) }`
- Now: inline in the button's `onClick` — images[0].src is guaranteed to exist, no guard needed

**`SizingAccordion` extracted** to its own file `SizingAccordion.tsx`.

---

### `src/app/(shop)/product/[slug]/SizingAccordion.tsx` — **New file**
Accordion for sizing/fit images. Uses CSS grid `grid-template-rows` animation:
```tsx
style={{ gridTemplateRows: open ? "1fr" : "0fr" }}
```
Animates height without knowing the content height. The inner div needs `overflow-hidden`.

---

### `src/app/(shop)/product/[slug]/ImageGallery.tsx`
- Removed empty state fallback — images are guaranteed to exist by the time this renders

---

### `src/app/(shop)/product/[slug]/page.tsx`
**`initialSelections` from all search params:**
- Was: only `color` param was passed
- Now: `const [{ slug }, { path, ...attrParams }] = await Promise.all([params, searchParams])` — destructures `path` out, all remaining params become `initialSelections`
- Filter to string values (TypeScript): `Object.entries(attrParams).filter((entry): entry is [string, string] => typeof entry[1] === "string")`
- `Promise.all([params, searchParams])` — awaits both in parallel instead of sequentially

**`categoryPath` removed from `Related`** — ProductGrid no longer needs it.

---

### `src/app/(shop)/category/[...path]/page.tsx`
- `Promise.all([params, searchParams])` for parallel awaiting
- `ProductSection` no longer takes `categoryPath`
- Thin `LoadMore.tsx` wrapper added, delegates to generic `LoadMoreProducts`

---

### `src/app/(shop)/category/[...path]/LoadMore.tsx` — **New file**
Thin wrapper: closes over `categoryId` and `filter`, passes `fetchPage` to generic `LoadMoreProducts`.

---

### `src/app/(shop)/sale/page.tsx`
- Added "All" filter option (`slug: null`)
- Active state fixed: `slug === null ? !filter : filter === slug` (was missing the "all" case)
- `href` for active filter now goes to `/sale` instead of toggling (clicking active filter = clear)
- Removed `export const dynamic = "force-dynamic"` — not needed

---

### `src/app/(shop)/sale/LoadMore.tsx` — **New file**
**`LoadMoreSaleProducts.tsx` — Deleted**
Same refactor as category: thin wrapper + generic `LoadMoreProducts`.

---

### `src/app/(bare)/checkout/page.tsx`
**`applyValidation` function extracted:**
- Was: inline logic in two places (on mount + on proceed), used `invalidItems: Set<string>` to visually dim items
- Now: `applyValidation(data)` removes non-existent items from cart and updates prices in place using `useUpdateCartPrice`
- Shows human-readable error: `"2 items removed · 1 item price updated"`

**`useUpdateCartPrice` added to cart:**
- Mutates price for a specific item in the Map
- Used when backend says price has changed since item was added

**Validation `useEffect` now has `[]` deps (runs once on mount), not `[items]`:**
- Validates on page load, not every time items change — avoids re-validating after the validation itself removes/updates items

**Promise chain → async/await:**
- `validateCart(items).then(...).catch(...)` → `async function validate() { try { await validateCart } catch {} }`

---

### `src/app/(bare)/order/success/page.tsx`
**Cart cleared on mount, not on countdown finish:**
- Was: `clear()` called inside the countdown `useEffect` at `seconds <= 0`
- Now: separate `useEffect(() => { clear() }, [])` fires immediately on mount
- Prevents cart from persisting if user navigates away before countdown ends

---

## Key Patterns & Decisions

### Price Display Logic (everywhere)
| Context | Condition | Display |
|---|---|---|
| ProductCard (list) | `sale && !variations.length` | Strikethrough + sale price |
| ProductCard (list) | else | Range (`min–max`) or single |
| Search panel | `sale && !variations.length` | Strikethrough + sale price |
| Search panel | `min === max` | Single price |
| Search panel | else | Range |
| ProductDetail | `!sku` | Range or single (product-level) |
| ProductDetail | `onSale` | Strikethrough + sale price |
| ProductDetail | else | `regularPriceCents` |

**Rule:** Variable products (have variations) never show strikethrough in list views. In detail view, once a variation is selected (`sku` resolves), strikethrough is safe.

### `SheetClose asChild` Pattern
Avoids nested button/anchor elements. Radix/shadcn `asChild` merges props + behaviour onto the child element:
```tsx
<SheetClose asChild>
  <Link href="...">Go somewhere</Link>
</SheetClose>
```
The Link closes the sheet when clicked. No extra DOM node, no `onClose` prop threading.

### Generic `LoadMoreProducts` Pattern
```ts
fetchPage: (page: number) => Promise<{ products: ProductListItem[]; hasNextPage: boolean }>
```
Each route provides its own `fetchPage` closure. The component doesn't know whether it's fetching category products or sale products.

### `availableByAttr` — Only for Unselected Attrs
Computing "available options for color given color is selected" doesn't make sense. Only compute available for attrs the user hasn't chosen yet. Once chosen, skip it — you already know what they picked.

### Debounce + Cleanup Pattern
```ts
useEffect(() => {
  if (!query.trim()) { reset(); return; }
  const timeout = setTimeout(async () => { ... }, 300);
  return () => clearTimeout(timeout); // cleanup cancels pending search
}, [query]);
```

### localStorage Key Convention
Changed from `type:brand` to `brand:type` (e.g. `${brandSlug}:cart`, `${brandSlug}:bookmarks`). Brand is the primary namespace since the same browser could have multiple brands.

### Unauthorized Handling in Providers
Both Cart and Bookmark providers now catch `Unauthorized` on sync and call `setLoggedIn(false)` rather than silently swallowing the error.

---

## JS/TS Gotchas

| Gotcha | Detail |
|---|---|
| `[] &&` renders `0` | Empty array is truthy. `[].length` is `0`. `0 &&` short-circuits to `0` which React renders. Use `arr.length > 0 &&` |
| `!arr` vs `!arr.length` | `![]` is `false` (arrays are truthy). `![].length` is `!0` = `true`. Use `.length` check |
| `!n` on a number | `!0` = `true`, `!1` = `false`. Safe for length/count checks |
| `!!value` | Double negation coerces to boolean. Used when TypeScript needs `boolean`, not just truthy |
| Optional chaining `?.` | Returns `undefined` if left side is nullish. `!undefined` = `true` |
| Short-circuit `attrSelected ? true : available.has(slug)` | Avoids calling `.has` on `undefined` when `available` doesn't exist for a selected attr |
| TypeScript non-null assertion `!` vs logical NOT `!` | Same character, two different operators. `salePriceCents!` = "I know this isn't null". `!flag` = logical not |
| `import type` | Compile-time only — erased at runtime. Use for types that don't need to exist as values |
| Sort comparator | Returns negative to put `a` first, positive to put `b` first, 0 for equal |
| CSS `grid-template-rows` animation | Animates from `0fr` to `1fr` for height-unknown expand/collapse. Inner div needs `overflow-hidden` |
| `Promise.all([params, searchParams])` | Next.js page params are Promises — await both in parallel to avoid sequential waterfall |

