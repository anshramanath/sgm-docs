# Learnings & What Was Built (June 27, 2026)

## What Was Built

### Provider Architecture
Three context providers wrap the app in `Providers.tsx`: `AuthProvider` → `CartProvider` → `BookmarkProvider`. Auth is outermost because cart and bookmark both read `loggedIn` from it.

### AuthProvider
- Calls `getUser()` on mount via `useEffect`. If user exists, sets `loggedIn: true`.
- Exposes `useLoggedIn()` and `useSetLoggedIn()` as named hooks.
- Private `useAuthContext()` helper sits between the provider function and the exported hooks — throws if used outside the provider.
- `loggedIn` starts `false` so no flash on initial render if signed out.

### CartProvider / BookmarkProvider
- State is a `Map<string, Item>` keyed by `slug:sku` (cart) or `slug` (bookmarks) — O(1) lookups vs O(n) array scans.
- Three `useEffect`s:
  1. `[loggedIn]` — runs `sync()`: reads localStorage, optionally fetches from DB, merges, sets state and `loaded: true`.
  2. `[items]` — writes to localStorage (guarded by `loaded`).
  3. `[items]` — debounced 800ms write to DB (guarded by `loaded && loggedIn`).
- `sync()` is async but the effect doesn't await it — fire and forget, runs in background, eventually calls `setItems`/`setLoaded`.
- 401 on any DB call (get or put) calls `setLoggedIn(false)` to propagate signed-out state across the app.

### Map State Pattern
- Maps are mutable — mutating in place keeps the same reference, React bails out (no re-render).
- Always do `new Map(prev)` to get a new reference, then modify the copy and return it.
- Serialise with `[...map.values()]` for localStorage/API (Maps aren't JSON-serialisable).
- Reconstruct with `mergeCartItems(local, db)` which iterates arrays and builds a fresh Map.

### Hook Pattern
Each provider exposes named hooks (e.g. `useAddToCart`, `useToggleBookmark`) that:
1. Call `useCartContext()` / `useBookmarkContext()` to get `setItems`.
2. Return a function (closure) the caller can invoke later.

The returned function is not a hook — only the outer function that calls `useContext` is a hook. The returned function captures `setItems` via closure and can be called anywhere.

### Cart Hooks
- `useAddToCart()` — adds item at qty 1, or increments if key already exists.
- `useRemoveFromCart()` — deletes by key.
- `useIncrementQty()` — increments qty; no-ops (returns `prev`) if key missing.
- `useDecrementQty()` — decrements qty; floors at 1, returns `prev` unchanged if already at 1.
- `useClearCart()` — replaces state with `new Map()`.
- Returning `prev` from a setState callback = same reference = React bails out = no re-render.

### Decrement Floor
`useDecrementQty` never deletes — floors at 1. UI also disables the decrement button at `quantity <= 1` with `disabled:opacity-30`. Prevents accidental deletion.

### Bookmark Hooks
- `useToggleBookmark()` — adds if missing, deletes if present. Handles both add and remove, so no separate `useRemoveBookmark` needed.
- `useIsBookmarked(slug)` — `map.has(slug)` returns boolean.
- `useBookmarkItems()` — `[...map.values()]` returns array.

### Explicit Field Construction
When setting a new item in a Map, fields are written out explicitly rather than spreading the input object. This ensures only the fields matching the type end up stored — spreading an input with extra fields (e.g. from a product detail page) would silently include them.

### localStorage Key Format
`${brandSlug}:cart` and `${brandSlug}:bookmarks` — brand slug first, so it's easy to identify entries per brand when inspecting localStorage across tabs.

### isActuallySignedIn Pattern
`HeaderIcons` receives `isSignedIn` as a prop from the server (prevents flash on initial render) and also calls `useLoggedIn()` from context. `isActuallySignedIn = isSignedIn && loggedIn` means:
- On initial mount: server value is authoritative (no flash).
- After mount: context value is live — a 401 from a provider sets `loggedIn: false`, which re-renders `HeaderIcons` and flips to "Sign In".

### API Error Handling Strategy
Defined per endpoint based on what the caller can do with the error:

| Endpoint | 401 | 500/503 | Default |
|---|---|---|---|
| `getCategories` | — | redirect `/try-again` | redirect `/try-again` |
| `getProducts` / `getSaleProducts` | — | throw | redirect `/try-again` |
| `getItem` | — | throw | redirect `/try-again` (404 → notFound) |
| `searchProducts` | — | throw | redirect `/try-again` |
| `validateCart` | — | throw | return `{data, status}` |
| `getCart` / `putCart` / `getBookmarks` / `putBookmarks` | throw "Unauthorized" | throw | redirect `/try-again` |
| `getOrders` / `createCheckoutSession` | redirect `/sign-in` | throw | redirect `/try-again` |

- `throw` → caller (server component) catches and shows inline error + retry button.
- `redirect` → Next.js redirects immediately, nothing bubbles.
- `throw "Unauthorized"` → providers catch it and call `setLoggedIn(false)`.

---

## React Concepts

### Re-renders
A component re-renders when its state changes, its props change, or its context changes. A re-render does NOT automatically re-render children — only components subscribed to the changed context or receiving changed props re-render.

### Object.is Comparison
React uses `Object.is` to decide whether state changed:
- Primitives: compared by value. `setState(5)` when state is already `5` → bail out.
- Objects/Maps/arrays: compared by reference. Mutating in place → same reference → bail out. `new Map(prev)` → new reference → re-render.

### Context Subscriptions
`useContext(SomeContext)` subscribes the component to that context. When the provider re-renders and passes a new `value` object, all subscribers re-render. The value object is recreated on every provider re-render (new reference), which is why all subscribers re-render even if the values inside are unchanged. `useMemo` can stabilise the reference.

### useMemo
`useMemo(() => value, [deps])` — caches the return value across re-renders. Only recomputes when deps change. Useful to prevent a new object reference from being created every render, which would cause unnecessary subscriber re-renders.

### useEffect Async Pattern
`useEffect` callback cannot be async. Use an inner async function and call it immediately:
```ts
useEffect(() => {
  async function doWork() { ... }
  doWork(); // don't await — effect callback must return cleanup or nothing
}, [deps]);
```

### Dependency Arrays
Only include what should trigger the effect. `[loggedIn]` for sync (re-sync when auth changes). `[items]` for writes (write whenever items change). Including unrelated values causes unnecessary effect runs.

### Functional setState
`setState(prev => next)` — React calls the callback with the current state. Whatever you return becomes the new state. Returning `prev` unchanged = no re-render. This is the correct pattern when the next state depends on the current state.

### never Type
`redirect()` and `throw` both have return type `never` in TypeScript — they never return normally. TypeScript doesn't require them in the function's return type because they terminate the execution path.

### Short-circuit Evaluation
`A && B` — if `A` is falsy, `B` is never evaluated. `A || B` — if `A` is truthy, `B` is never evaluated. After `e instanceof Error`, TypeScript narrows `e` to `Error` so `.message` is safe without optional chaining.

### De Morgan's Law
`!(A && B) === !A || !B` and `!(A || B) === !A && !B`. An early return `if (!loaded || !loggedIn) return` is equivalent to only proceeding when `loaded && loggedIn`.

---

## JavaScript Concepts

### Property Access & Calls
`obj.method()` — `.` looks up the value at key `method`, `()` invokes it. The dot itself does nothing to call; `()` is what triggers execution. Arguments go inside `()`.

### Destructuring
`const { key } = obj` — extracts `obj.key` into a local variable named `key`. To rename: `const { key: newName } = obj`. Key name must match the object's actual key.

### Shorthand Properties
`{ loggedIn, setLoggedIn }` is shorthand for `{ loggedIn: loggedIn, setLoggedIn: setLoggedIn }`. Key and variable name are the same.

### Spread in Object Literals
`{ ...existing, quantity: existing.quantity + 1 }` — copies all fields from `existing`, then `quantity` is written again after, overwriting the spread's version. Last key wins.

### Map vs Array
- `Map.has(key)` → boolean, O(1).
- `Map.get(key)` → value or `undefined`, O(1).
- `Map.set(key, value)` → sets entry, O(1).
- `Map.delete(key)` → removes entry, no-op if missing, O(1).
- `Map.values()` → returns a `MapIterator`, not an array. Spread to convert: `[...map.values()]`.

### Omit<T, K>
TypeScript utility type. Removes key `K` from type `T`. `Omit<CartItem, "quantity">` = CartItem without the quantity field — useful when the function provides the value itself.

### Idempotent vs No-op
- No-op: does nothing at all.
- Idempotent: calling multiple times produces the same result as calling once. A no-op is idempotent, but idempotent operations can still do something (e.g. writing the same value to DB — the write happens, but the state is unchanged).

### Absolute Imports
`@/components/foo` vs `../components/foo`. Absolute imports don't break when files are moved. Always use absolute when `@/` is configured in tsconfig.
