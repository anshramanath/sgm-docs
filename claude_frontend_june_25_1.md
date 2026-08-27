# Frontend Notes (June 25, 2026)

## Stack

- **Next.js App Router** — server components by default, `"use client"` to opt into client rendering
- **TypeScript** — strict types throughout
- **Tailwind CSS** — utility classes, custom design tokens via CSS variables
- **Supabase** — auth (session management, JWT tokens)
- **Stripe** — checkout via redirect to hosted payment page

---

## Architecture

### Route Groups

- `(shop)` — main storefront with navbar, ticker, full layout
- `(bare)` — minimal layout, no navbar (checkout, account, signin)
- Root — error pages (404, try-again, error.tsx) and root layout

### Server vs Client

All components are server by default. Client components (`"use client"`) are used only when browser APIs or interactivity is needed:
- Providers (cart, bookmarks, auth state)
- Product detail (variant selection, add to bag)
- Checkout page (session check, proceed button)
- Header icons (search, bag count, bookmark count)
- Load more buttons

---

## Environment Variables

All server-side only — no `NEXT_PUBLIC_` prefix:

```
SUPABASE_URL=
SUPABASE_ANON_KEY=
SERVER_BASE_URL=      # backend API base URL
BRAND_SLUG=           # which brand this deployment serves
```

Changing `BRAND_SLUG` switches which brand the entire site serves.

---

## Brand System (`src/lib/brand.ts`)

`"use server"` file — never accessible client-side. Reads `BRAND_SLUG` from `process.env` and returns only the config for that brand. The full `BRANDS` map is unexported so no deployment can see other brands' configs.

```ts
export async function getBrand() {
  return BRANDS[process.env.BRAND_SLUG as keyof typeof BRANDS];
}
```

Each brand object contains: `slug`, `name`, `description`, `logo`, `hero`, `accent`, `heroCopy`, `editorial`.

Server components call `const brand = await getBrand()` — Next.js deduplicates the call within a render so it only executes once per request even if called from multiple components.

The `accent` color is injected into the root layout as a CSS variable (`--color-brand`) so client components can reference `var(--color-brand)` without importing brand config.

---

## API Layer (`src/lib/api.ts`)

`"use server"` — all exported functions are server actions, callable from both server components and client components.

`BRAND_SLUG` and `SERVER_BASE_URL` are read from `process.env` internally. No caller ever passes a brand slug — it's locked to the deployment.

### Helpers

**`apiFetch`** — GET requests with query params, `revalidate: 60` cache. Returns a synthetic 503 `Response` on network failure so callers always get a uniform `Response` object.

**`authedFetch`** — POST/PUT with Bearer token from Supabase session. Returns 401 `Response` if no session, 503 on network failure.

### Return types

- Public catalog functions → `Promise<ApiResponse<T>>`
- `validateCart` → `Promise<ValidateCartItem[]>` (raw array, no wrapper)
- `createCheckoutSession` → `Promise<CheckoutResponse>` (union type, branches on `res.ok`)

### Why synthetic error Responses

Instead of returning `null` or throwing, network failures return `new Response(JSON.stringify({ success: false, error: "..." }), { status: 503 })`. This means callers always get a real `Response` with `.ok`, `.status`, and `.json()` — no null checks needed.

---

## Type System (`src/lib/types.ts`)

### Key decisions

- `ListVariation.slug` is the unique identifier (not `id`) — used as React key and in URL params
- `Variation.sku` is the unique identifier for a product variation
- `CartAttribute` stores `{ name, option, slug }` — `option` for display, `slug` for URL reconstruction
- `OrderItem` shows `attribute` as a pre-formatted string (not structured) — intentional, SKU not shown in order history
- `ApiResponse<T>` is a discriminated union: `{ success: true; data: T } | { success: false; error: string }`
- `CheckoutResponse` is a union: `{ success: true; url: string } | { success: false; items: ValidateCartItem[] }`

---

## Error Pages

| Page | File | Trigger | Action |
|------|------|---------|--------|
| 404 | `app/404/page.tsx` | `redirect("/404")` from code | Back to Home |
| Not Found | `app/not-found.tsx` | Invalid URL (auto by Next.js) | Back to Home |
| Try Again | `app/try-again/page.tsx` | `redirect("/try-again")` on 5xx | Back to Home (retries via home) |
| Error | `app/error.tsx` | Unhandled render exception | Try Again (calls Next.js `reset`) |

### Why separate 404 and not-found

`not-found.tsx` inherits the shop layout (navbar) when triggered from within a shop route — by design. Our intentional not-found calls use `redirect("/404")` which renders under the root layout only, no navbar. The auto not-found (invalid URL) uses `not-found.tsx` — acceptable since it's rare.

### Error type reasoning

- **404** — permanent, resource doesn't exist → go home
- **5xx / try-again** — transient, server/DB failure → retry (going home re-requests the same server, either it works or you land on try-again again)
- **error.tsx** — unexpected React render exception → `reset()` retries the same page in place, URL never changes

### HTTP status semantics used

- `401` Unauthorized — no session
- `404` Not Found — resource doesn't exist  
- `503` Service Unavailable — network error, upstream unreachable
- `5xx` → redirect to try-again

---

## Cart & Bookmarks (`src/components/providers/`)

Both use the same pattern:
1. Load from `localStorage` on mount
2. Fetch from DB (server action)
3. Merge — DB wins on conflict
4. Persist to `localStorage` on every change (after initial load)
5. Debounced DB sync (800ms) on every change

localStorage keys: `"cart"` and `"bookmarks"` — no brand suffix since each deployment is one brand.

Cart item key: `${productSlug}:${sku}` — used for dedup and quantity updates.

---

## Checkout Flow

1. `checkout/page.tsx` — client component, validates cart on mount via `validateCart`
2. Invalid items (not exists or price changed) are flagged with `${productSlug}:${sku}` key
3. On proceed: session check → `createCheckoutSession` → redirect to Stripe URL
4. On Stripe 200: `{ url: string }` → `window.location.href = res.url`
5. On Stripe 4xx: returns `ValidateCartItem[]` → update `invalidItems` set, user sees flags

---

## Navigation & URL Structure

- `/` — home
- `/category/[...path]` — category page, path is slug chain (e.g. `/category/sunglasses/sport`)
- `/product/[slug]` — product detail
- `/sale` — sale page
- `/checkout` — checkout
- `/account` — order history, account details
- `/signin` — auth
- `/404` — not found
- `/try-again` — server error

Product variation URLs use `?color=<variation-slug>` query param. `CartAttribute.slug` stores the variation slug for reconstructing these URLs from cart items.

---

## Concepts Covered

### Next.js
- Server vs client components — server by default, async allowed
- `"use server"` — marks async exports as server actions (callable from client via RPC)
- `"server-only"` package — prevents client imports entirely (build error)
- `generateMetadata` — async alternative to static `metadata` export
- `notFound()` — triggers nearest `not-found.tsx`, inherits parent layouts
- `redirect()` — throws internally, Next.js intercepts; cannot be used in `not-found.tsx`
- `error.tsx` — React error boundary for unhandled render exceptions; receives `reset` to retry
- `revalidate: 60` — Next.js fetch cache, only caches 2xx responses (bad responses never cached)
- Route groups `(name)` — group routes under a shared layout without affecting the URL

### Web APIs
- `Response` object — `.ok`, `.status` are properties; `.json()` reads and parses the body stream (one-time read)
- `NextResponse` extends `Response` with Next.js helpers; `NextResponse.json()` is shorthand for `new Response(JSON.stringify(...), { headers: ... })`
- `window.location.href` — full browser navigation, reloads the app shell
- `<Link href>` — Next.js client-side navigation, swaps page content without full reload

### TypeScript
- `as const` — narrows object/array types to their literal values
- Discriminated unions — `{ success: true; data: T } | { success: false; error: string }` — TypeScript narrows the type after checking `success`
- `keyof typeof` — gets the union of keys from an object type

### HTTP
- `401` Unauthorized, `404` Not Found, `503` Service Unavailable
- `res.ok` — true for 200–299 status codes
- POST vs GET — GET for reads (cacheable), POST for mutations or when sending a body
