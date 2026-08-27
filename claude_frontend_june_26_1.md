# Learnings & Architecture Notesm (June 26, 2026)

## What We Built

### `src/lib/api.ts`
All server-side API calls. Three fetch helpers, all public endpoints return data directly (not wrapped in `ApiResponse`).

**Fetch helpers:**
- `apiFetch` — GET with query params, 60s revalidation
- `publicPostFetch` — unauthenticated POST (no JWT, no cache)
- `authedFetch` — checks user exists, attaches Bearer token, POST/PUT

**Error handling pattern:**
```ts
if (!json.success) {
  switch (res.status) {
    case 404: notFound();
    case 500: redirect("/try-again");
    case 503: redirect("/try-again");
    default:  redirect("/try-again");
  }
}
return json.data;
```
All switch branches return `never` → TypeScript narrows `json` to the success branch after the block, so `json.data` is typed correctly without an extra assertion.

**Special cases:**
- `validateCart` — only redirects on 500/503. 404/409/422 fall through and return `{ data, status }` so the cart UI can show validation errors.
- `createCheckoutSession` — same pattern. Returns `CartValidationResult | CheckoutUrl`.
- `authedFetch` — returns synthetic 401 response if `getUser()` returns null, before the network call. This gives the same switch-based handling at the call site.

**Network errors:**
Each fetch helper catches and returns a synthetic `Response` with `{ success: false, message: "Network error!" }` and status 503. This keeps error handling uniform — callers never need to special-case thrown exceptions.

---

### `src/lib/types.ts`
```ts
type ApiResponse<T, E = never> =
  | { success: true; data: T }
  | { success: false; message: string; data?: E };

type ValidateCartItem = { productSlug: string; sku: string; exists: boolean; priceCents: number | null; priceChanged: boolean };
type CartValidationResult = { data: ValidateCartItem[]; status: number };
type CheckoutUrl = { url: string };
type SyncedResponse = { synced: number };
```
`ApiResponse` takes an optional second generic `E` for the failure `data` field (used in validate-cart and checkout where the server returns partial data on 4xx).

---

### `src/lib/auth.ts`
```ts
getToken()   → string | null   // just the JWT access token
getUser()    → User | null     // safe, no redirect
requireUser() → User           // redirects to /sign-in if null
```
`getToken` and `getUser` are separate because authenticated API calls only need the token, but some pages need the full user object. `requireUser` is just `getUser` + redirect — kept simple.

`session?.access_token ?? null` — optional chain + nullish coalesce handles the case where no session exists without needing an explicit null check.

---

### `src/lib/utils.ts`
Category utilities merged in (was `categoryUtils.ts`):

```ts
findCategoryId(tree, segments)  // walks tree by slug array, returns leaf id
collectLeaves(tree)             // returns flat Record<slug, { id, name, path }>
```

`collectLeaves` uses a private helper that mutates a shared result object through reference. The public function takes only `tree` and creates the result — callers don't need to pass an accumulator.

---

### `src/lib/brand.ts`
Brand config is a `const` object keyed by slug — `getBrand()` looks up by `process.env.BRAND_SLUG`.

Each brand has:
- `categoryImages: string[]` — 5 placeholder images (one per category tile)
- `editorial: { body: string; image: string }[]` — 2 editorial slots with copy and image

These drive the homepage without any product image fetching.

---

### `src/app/(shop)/page.tsx`
Homepage structure:
1. **Hero** — static copy + hero image from `brand.ts`
2. **Top Categories** — maps over `brand.categoryImages`, uses `leaves[i % leaves.length]` for the category link/name
3. **Best Sellers** — async component in `<Suspense>`, fetches products. Only this section blocks.
4. **Editorial Split** — maps over `brand.editorial`, uses `leaves[(4 + i) % leaves.length]`

`getCategories()` is called in both Navbar and page — Next.js deduplicates server fetches so it's only one request.

---

## TypeScript Lessons

### Narrowing with switch
`if (!json.success && res.status === 500)` does **not** narrow `json` — TypeScript can't track the relationship between two variables. A switch where every branch returns `never` does work:
```ts
if (!json.success) {
  switch (res.status) {
    case 500: redirect("/try-again"); // never
    default:  redirect("/try-again"); // never
  }
}
// json is now narrowed to { success: true; data: T }
return json.data; // ✓
```

### `never` after redirect/notFound
`redirect()` and `notFound()` both return `never`. Code after them is unreachable — TypeScript knows this, so you don't need a `return` or `break` after them in a switch.

### Two-generic ApiResponse
```ts
ApiResponse<ValidateCartItem[], ValidateCartItem[]>
```
The second generic types the optional `data` field on the failure branch. Even with this, the field is typed as `E | undefined` (because it's `data?: E`), so you still need `?? []` when accessing it.

---

## Next.js Lessons

### Server fetch deduplication
Next.js deduplicates identical `fetch()` calls within a single render. `getCategories()` in Navbar + `getCategories()` in page = one network request.

### Suspense boundary
Only content inside the boundary is held; everything outside renders immediately. So wrapping just `<BestSellers />` in `<Suspense>` means the hero, categories, and editorial render instantly while products stream in.

### redirect() / notFound() in server components
Both throw internally — they work from server components, server actions, and route handlers. In a switch statement, they terminate the branch the same way `return` would.

### Edge runtime (middleware)
`process.env.NEXT_PUBLIC_*` is required in middleware because it runs on the edge runtime before the Node.js environment is available. This is the valid exception to the no-`NEXT_PUBLIC_` rule for server code.

---

## Patterns & Conventions

### Modulo for safe array access
```ts
leaves[i % leaves.length]          // wraps around, never out of bounds
brand.categoryImages[i % brand.categoryImages.length]
```
Use when iterating a fixed-length array of display slots against a variable-length data array. Constraint: the data array must have at least 1 element.

### Brand-driven rendering
Use `brand.categoryImages.map(...)` (not `Array.from({ length: 5 })`). The brand defines the slot count — data wraps to fill it with `%`.

### Error message convention
Synthetic error responses use `message` (not `error`) to match `ApiResponse`, with `!` suffix: `"Network error!"`, `"Unauthorized!"`.
