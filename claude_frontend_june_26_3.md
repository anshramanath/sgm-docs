# What We Built & Learned (June 26, 2026)

## Brand System

`NEXT_PUBLIC_BRAND_SLUG` in `.env` makes the slug available on both server and client, turning `getBrand()` into a synchronous function. No more `await getBrand()`, no prop drilling the brand into providers or checkout. All image paths live under brand subfolders in `public/` (`prosport-sunglasses/hero.jpg`, etc.).

The brand color (`--color-brand`) is set at runtime in `layout.tsx` as an inline CSS variable — not hardcoded in `globals.css`. This means every brand gets its own accent color without any conditional logic. `--color-sale` was removed entirely since it was always the same value.

---

## API Layer (`api.ts`)

Three internal fetch helpers:
- `apiFetch` — public catalog, GET with cache (`revalidate: 60`)
- `publicPostFetch` — unauthenticated POST (validate cart)
- `authedFetch` — authenticated POST/PUT, injects JWT from `getToken()`

`authedFetch` calls `getUser()` first. If null, it returns a synthetic 401 response before hitting the network.

### Error Handling Strategy

The strategy is intentional per endpoint:

| Function | 401 | 500/503 | Default |
|---|---|---|---|
| `getCategories` | — | redirect `/try-again` | redirect |
| `getProducts` | — | throw | redirect |
| `getSaleProducts` | — | throw | redirect |
| `getItem` | 404 → notFound() | throw | redirect |
| `searchProducts` | — | throw | throw |
| `validateCart` | — | throw | return `{ data: [], status }` |
| `getCart` / `getBookmarks` | throw "Unauthorized" | throw | redirect |
| `putCart` / `putBookmarks` | throw "Unauthorized" | throw | redirect |
| `getOrders` | redirect `/sign-in` | throw | redirect |
| `createCheckoutSession` | redirect `/sign-in` | throw | return validation data |

**Why this split:**
- `getCategories` always redirects — the page can't render without it
- `getProducts` / `getSaleProducts` / `getItem` throw on 500/503 so the page can catch and show an inline retry button — transient errors should recover on retry
- `searchProducts` throws for all cases — search failure should show a message in the panel, not redirect
- Cart/bookmark throw on 401 — caught by providers to flip auth state silently; redirect on unknown because unaccounted errors could be serious
- `getOrders` / `createCheckoutSession` redirect on 401 — auth-gated pages, redirect is expected
- Default always redirects for unaccounted errors — we don't know if they're safe to ignore

`redirect()` and `throw` both produce `never` in TypeScript — neither needs to be in the return type because those branches never reach a `return` statement.

---

## Providers

### Pattern

All three providers follow the same structure:
1. Type definition
2. `createContext(null)` — honest about missing provider, no fake defaults
3. Helper functions (merge)
4. Provider component
5. Private `useXContext()` with null guard — throws if used outside provider
6. Individual exported hooks

**Why `createContext(null)`:** The initial value is meaningless since the real value is set in the component. `null` makes it explicit that using the hook outside the provider is an error, caught immediately by the null guard.

**Why individual hooks instead of one `useCart()`:** Explicit imports make it clear what each component actually uses. Each hook is self-contained and named after what it does.

### AuthProvider

```ts
const [loggedIn, setLoggedIn] = useState(false);
useEffect(() => {
  async function check() {
    try {
      const user = await getUser();
      if (user) setLoggedIn(true);
    } catch {}
  }
  check();
}, []);
```

Checks session on mount. `getUser()` validates the Supabase session from cookies. Empty `catch` is intentional — auth check failure means the user stays as a guest, which is fine.

### CartProvider & BookmarkProvider

**Map state instead of arrays:**
- `Map<string, CartItem>` keyed by `${productSlug}:${sku}`
- `Map<string, BookmarkedItem>` keyed by `productSlug`
- O(1) lookups (`map.get`, `map.has`, `map.delete`) vs O(n) `find`/`some`/`filter`
- Merge functions return a Map directly — no intermediate array conversion

**Three useEffects, one dep each:**
```ts
useEffect(() => { sync(); }, [loggedIn]);         // re-sync when auth changes
useEffect(() => { localStorage.set... }, [items]); // persist on every change
useEffect(() => { putCart/putBookmarks... }, [items]); // debounced DB write
```

`loggedIn` and `loaded` are not in the localStorage/DB effect deps because `items` always changes alongside them — the sync effect owns the `loggedIn` dependency, and its changes cascade through `items`.

**localStorage keys:** `${brandSlug}:cart`, `${brandSlug}:bookmarks` — brand first for readability when inspecting across multiple brands.

**sync() single exit point:**
```ts
let dbItems = [];
if (loggedIn) {
  try { dbItems = await getDB(); } catch (e) {
    if (e.message === "Unauthorized") setLoggedIn(false);
  }
}
setItems(merge(localItems, dbItems)); // merge with [] = returns local unchanged
setLoaded(true);
```

**Explicit field construction on add:**
```ts
next.set(key, {
  productId: item.productId,
  productSlug: item.productSlug,
  ...
});
```
No spread — prevents extra fields from callers leaking into the Map, localStorage, and the DB.

**Cart hooks:**
- `useAddToCart` — takes `Omit<CartItem, "quantity">`, increments if exists, adds at 1 if new
- `useRemoveFromCart(item)` — deletes by key
- `useIncrementQty(item)` — +1
- `useDecrementQty(item)` — -1, floors at 1 (never deletes)
- `useClearCart` — `new Map()`
- `useCartItems()` → `[...map.values()]`
- `useCartCount()`, `useCartTotal()` → iterate map values

**Bookmark hooks:**
- `useToggleBookmark(item)` — adds if not present, removes if present. One function covers both directions since bookmarks are binary
- `useBookmarkItems()` → `[...map.values()]`
- `useIsBookmarked(slug)` → `map.has(slug)` — O(1) boolean

---

## Auth State & Re-render Flow

### The Problem
`Navbar` is a server component — it runs `getUser()` once at request time and passes `isSignedIn` as a prop to `HeaderIcons`. If the user's session expires during a background cart sync (401), the server component doesn't know. `HeaderIcons` would still show the account icon even though the user is effectively signed out.

### The Solution
```ts
// HeaderIcons
const loggedIn = useLoggedIn();
const isActuallySignedIn = isSignedIn && loggedIn;
```

- `isSignedIn` (prop from server): prevents flash on initial mount — the server already knows auth state before the page loads
- `loggedIn` (context): updates live when a 401 is caught in the providers

When a provider catches a 401:
```ts
} catch (e) {
  if (e instanceof Error && e.message === "Unauthorized") setLoggedIn(false);
}
```
This flips `loggedIn`, `AuthProvider` re-renders with a new context object, `HeaderIcons` (subscribed via `useLoggedIn()`) re-renders, `isActuallySignedIn` becomes `false`, the account icon switches to "Sign In".

**Why not redirect on cart/bookmark 401:** Cart and bookmark syncs happen on public pages. Redirecting to `/sign-in` mid-browse would feel like the page is auth-gated. Silent state update is less disruptive.

**Why redirect on orders/checkout 401:** Those pages are explicitly auth-gated. Redirect is expected, and navigating back to the shop triggers a full server render which picks up the correct auth state.

---

## React Concepts

### Re-renders & References

React uses `Object.is` for all state comparisons:
- **Primitives** (`boolean`, `number`, `string`): same value = no re-render
- **Objects/Arrays/Maps**: same reference = no re-render, different reference = re-render (even if contents are identical)

This is why we always do `new Map(prev)` — mutation changes contents but not the reference, so React wouldn't see a change. `new Map(prev)` creates a new reference.

```ts
// Re-render: new reference
setItems(new Map(prev));

// No re-render: same reference despite new contents
prev.set(key, value); // ← mutation
setItems(prev);
```

### Context Re-renders

`useContext` creates a subscription. When the context value reference changes (because the provider re-rendered with new state), all subscribers re-render. Only components that called `useContext` re-render — not every component in the subtree.

The chain:
1. `setLoggedIn(false)` — state change
2. `AuthProvider` re-renders — new object `{ loggedIn: false, setLoggedIn }` passed to `AuthContext.Provider`
3. React sees new context object reference
4. All `useLoggedIn()` / `useSetLoggedIn()` callers re-render

If `AuthProvider` had non-context state, changing it would still create a new context object and re-render all subscribers unnecessarily. `useMemo` on the context value prevents this — caches the object reference and only recomputes when actual deps change.

### Hooks

Hooks are functions that call React internals (`useState`, `useContext`, `useEffect`, etc.). They can only be called at the top level of a component or inside another hook — never inside conditionals, loops, or event handlers.

**Why hooks return functions:**
```ts
export function useAddToCart() {
  const { setItems } = useCartContext(); // ← hook call at top level
  return (item) => { setItems(...) };   // ← returned function can be called anywhere
}
```
The hook call captures context at the top level. The returned function closes over `setItems` and can be called inside event handlers freely.

### useEffect Async Pattern

`useEffect` callbacks cannot be async (they must return void or a cleanup function, not a Promise). Use an inner async function:

```ts
useEffect(() => {
  async function sync() { ... }
  sync(); // fire and don't await — useEffect continues
}, [dep]);
```

`sync()` is called but not awaited. The `useEffect` returns immediately. `loaded` state guards renders that depend on the async result completing.

### .then() vs async/await

Same thing, different syntax:
```ts
// .then() — good for single promise
getUser().then((user) => { if (user) setLoggedIn(true); }).catch(() => {});

// async/await — better for multiple sequential steps
async function check() {
  try {
    const user = await getUser();
    if (user) setLoggedIn(true);
  } catch {}
}
```

### useMemo vs useCallback

- `useMemo(() => value, [deps])` — caches a value, recomputes only when deps change
- `useCallback(fn, [deps])` — caches a function reference (same as `useMemo(() => fn, [deps])`)

`useCallback` inside context providers is useless without `React.memo` on consumers — the context object reference changes on every state change anyway, causing all consumers to re-render regardless of whether the function reference stayed the same.

---

## Next.js

### Server vs Client Components

Server components run on the server at request time — no state, no effects, no browser APIs. They can be async and call APIs/databases directly. Client components (`"use client"`) run in the browser and can use hooks and event handlers.

**Re-renders:** Only client components re-render. Server components don't respond to client state changes — they only re-run on navigation or refresh.

### `redirect()` and `notFound()`

Both throw special internal errors (`NEXT_REDIRECT`, `NEXT_NOT_FOUND`). TypeScript treats them as `never` — branches that call them don't need to be in the return type. Next.js's runtime catches these and performs the navigation.

### Server Actions (`"use server"`)

Functions marked `"use server"` run on the server but can be called from client components. Each call is an HTTP POST round trip — an extra network request on top of whatever the function does internally. Worth knowing when deciding whether to use server actions vs API routes.

### Middleware

`proxy.ts` intercepts every request and refreshes the Supabase session via `createServerClient`. The refreshed session (new cookies) is forwarded to the server. This is why Supabase access tokens (1 hour TTL) are silently refreshed — the middleware handles it on every page load as long as the refresh token (60 day TTL) is still valid.

---

## Supabase Auth

- **Access token (JWT):** 1 hour TTL, refreshed automatically by middleware
- **Refresh token:** 60 day TTL, stored in cookies. Once expired, session is gone and user must sign in again
- **Storage:** Cookies (not localStorage). `getUser()` reads and validates the session from cookies. Returns `null` if invalid or expired.

---

## localStorage

- Just strings — `JSON.stringify` / `JSON.parse` for objects
- Brand-namespaced keys: `${brandSlug}:cart`, `${brandSlug}:bookmarks`
- Survives tab close and browser restarts (unlike sessionStorage)
- Written synchronously — even if the DB sync is cancelled on unmount (tab close), localStorage has the data for the next session
- DB sync is debounced 800ms — unmounting cancels the timeout via cleanup, but localStorage already has it
