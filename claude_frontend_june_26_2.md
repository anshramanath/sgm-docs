# Architecture & Decisions (June 26, 2026)

## Brand System

`NEXT_PUBLIC_BRAND_SLUG` is a public env var that identifies which brand the storefront serves. Making it `NEXT_PUBLIC_` was a deliberate choice — the slug is not sensitive (it's implied by the logo, domain, and content), and it unlocks sync access everywhere.

`getBrand()` in `src/lib/brand.ts` is a synchronous function. It does a direct lookup into a static `BRANDS` map using `process.env.NEXT_PUBLIC_BRAND_SLUG` and throws if the slug is unknown. Because it's sync and has no server-only dependencies, it can be called from anywhere — server components, client components, module scope.

Brand accent color is set as `--color-brand` CSS variable on `<body>` in `src/app/layout.tsx`. All components reference it via `var(--color-brand)` or the utility classes `.text-brand`, `.bg-brand`, `.border-brand`, `.decoration-brand` defined in `globals.css`. No component receives accent as a prop.

## API Layer (`src/lib/api.ts`)

All exported functions return data directly and never return an `ApiResponse` wrapper. On error they throw (triggering Next.js error boundaries) or redirect (via `notFound()` / `redirect()`). Callers never check `.success` or `.data`.

`ApiResponse<T>` is still used internally to type raw JSON from the backend before extracting `.data`. It does not appear in any function's return type.

`BRAND_SLUG` is read once at module scope from `process.env.NEXT_PUBLIC_BRAND_SLUG` and included automatically in every API call. Callers never pass it.

`authedFetch` redirects to `/sign-in` on 401. Client components (`CartProvider`, `BookmarkProvider`) guard calls with `if (loggedIn)` to avoid unwanted redirects.

## Category Tree

`collectLeaves` in `src/lib/utils.ts` is the single utility for working with the category tree. It returns `Record<path, LeafEntry>` where:
- Key is the full slash-joined slug path (e.g. `"brands/bikerarmour/photochromic"`)
- Value is `{ id, name, path, breadcrumbs: string[] }`

Only leaf nodes (no children) have products. The full-path key supports the same slug appearing at different tree levels. `breadcrumbs` carries the human-readable name path so pages never need to re-traverse the tree.

`findCategoryId` and `findPathNodes` were removed — `collectLeaves` replaces both.

## Breadcrumb Pattern

- **Category page**: Home is a link. All breadcrumb segments are plain text.
- **Product page**: Home is a link. The last breadcrumb (leaf category) is a link. Intermediate segments are plain text. The product name is a separate async `ProductName` component wrapped in `Suspense` with a skeleton fallback.

## Providers

`CartProvider` and `BookmarkProvider` call `getBrand().slug` at module scope for localStorage namespacing (`cart:${brandSlug}`, `bookmarks:${brandSlug}`). No prop drilling — `Providers.tsx` takes only `children`.

## Page Patterns

Server pages that only need brand data are synchronous — `getBrand()` is called inline with no `await`. Pages that also fetch API data (categories, products) stay `async` for those calls only.

`generateMetadata` functions follow the same rule: sync if brand-only, async if they also fetch API data.

Client pages (checkout) call `getBrand()` at module scope and use `useCart()` / server actions directly. No server wrapper needed.

## Color System

One accent color: `--color-brand`, set at runtime from `brand.accent`. No `--color-sale` or separate sale color — sale badges, sale prices, errors, and status indicators all use brand color. The CSS variable is set in `layout.tsx` and never hardcoded in `globals.css`.

## Modulo Pattern for Category Images

`brand.categoryImages` drives slot count on the homepage. Leaves are accessed via `leaves[i % leaves.length]` so the grid always fills even if there are fewer leaf categories than image slots. Same leaf can appear twice — `key={i}` not `key={leaf.path}` to avoid React key collisions.
