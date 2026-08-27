# Built & Learned (June 17, 2026)

## Auth Flow

### AuthProvider
- Simple React context holding `loggedIn: boolean` and `setLoggedIn`
- No `useEffect`, no Supabase client — just state
- `useLoggedIn()` returns the boolean, `useSetLoggedIn()` returns the setter
- Initialized to `false` — no need to check session on load because `getCart`/`getBookmarks` are server actions that check auth via cookies independently

### Why not `onAuthStateChange`?
- `SIGNED_IN` only fires when the Supabase **browser client** handles sign-in
- When sign-in is a server action, the browser client never fires the event
- `INITIAL_SESSION` fires on every page load regardless of auth state — using it without checking the session would set `loggedIn = true` even when logged out, causing `SIGNED_IN` to be a no-op later (same value, no re-render)

### Sign-in
- `signIn` server action no longer calls `redirect()` — returns `null` on success
- `SignInForm` calls `setLoggedIn(true)` then `router.push("/")` after no error
- `setLoggedIn(true)` triggers the cart/bookmark sync immediately; providers don't unmount on navigation so the sync runs through the redirect
- By the time the user lands on `/`, items are already populated

### Sign-out
- `SignOutButton` is a client component — calls `setLoggedIn(false)`, then `router.push("/")`, then `await signOut()`
- Navigation happens **before** `signOut()` — if you await signOut first, Next.js refetches the account page's server components during soft navigation, `requireUser()` finds no session, and redirects to `/signin`
- `router.push()` is non-blocking so `signOut()` still runs concurrently

---

## Cart & Bookmark Sync

### Strategy
- Both providers follow identical pattern: localStorage as local state, Supabase DB as remote state
- On mount: read localStorage → try `getCart`/`getBookmarks` → if DB returns data, merge → write merged back to DB → set state
- Merge rule: **DB wins on same slug** — no quantity summing
- `loaded` state gates localStorage write and debounced PUT to prevent writing empty initial state

### Sync trigger (`loggedIn` dependency)
- Sync effect depends on `[loggedIn]`
- On mount: runs once with `loggedIn = false` (handles already-logged-in case since server action checks cookies)
- On sign-in: `setLoggedIn(true)` → effect re-runs → fetches DB and merges
- On sign-out: `setLoggedIn(false)` → effect re-runs → DB returns null → falls back to localStorage
- Setting state to same value = no re-render = no effect re-run (React `Object.is` comparison)

### Debounce vs immediate PUT
- Debounced PUT (800ms) handles normal add/remove interactions
- Immediate fire-and-forget PUT in the sync function ensures DB is written right after merge
- Without the immediate PUT: if user navigates within 800ms, the debounce cleanup cancels the timer → DB never written
- Since providers are in root layout they don't unmount on navigation, so debounce timers do persist — but the immediate PUT is still safer for the sign-in sync case

### Brand scoping
- All queries scoped with `brand_slug` — `getCart(BRAND_SLUG)`, `getBookmarks(BRAND_SLUG)`
- Missing `.eq("brand_slug", brandSlug)` on the bookmarks GET endpoint caused cross-brand data pollution: all brands' rows merged into current brand's state, then PUT wrote everything back under the wrong brand

---

## Providers Architecture

- `Providers` lives in root `app/layout.tsx` — never unmounts across navigations
- This means: debounce timers persist, state persists, sync triggered by `setLoggedIn` runs through navigation
- Checkout page can access cart context because providers are at root (not scoped to `(shop)` layout)

---

## Category Filters

- Filter slugs: `under-15`, `15-25`, `25-plus`, `sale`
- Frontend passes `?filter=<slug>` as a URL param
- Backend `FILTER_MAP` translates slug → `{ minPrice, maxPrice, sale }` — price logic never exposed in URL
- `<Suspense key={filter ?? "all"}>` forces remount on filter change, preventing Next.js from reusing the cached component

---

## Sale Page
- Separate `/api/public/sale` endpoint — always filters `.eq("sale", true)`, no category required
- Price filters available but no "Sale" filter chip (already a sale page)
- `getSaleProducts` in `api.ts` calls this endpoint
- `LoadMoreSaleProducts` component mirrors `LoadMoreProducts` but uses `getSaleProducts`

---

## Search Panel
- `/api/public/search` endpoint: `GET` with `brandSlug` + `q`, uses `.ilike("name", "%q%")`, returns up to 6 results
- 300ms debounce before firing
- During debounce: keep showing previous results (featured or last search) to avoid flash
- After debounce: skeleton grid (6 cards with `animate-pulse`) while loading
- If no results: "No results for X" message
- `displayed = query.trim() ? results : featured`

---

## Panels (Cart, Saved, Search)
- All panels accept `onClose` prop
- Product links and checkout links inside panels call `onClose()` on click, closing the sheet before navigating
- Prevents panel staying open awkwardly after navigation

---

## Badge Animations
- Cart and bookmark count badges use `key={count}` / `key={bookmarks.length}`
- Changing `key` forces React to remount the element, re-triggering CSS entry animation
- `badge-pop` keyframe: scale 0.5 → 1.25 → 1 over 200ms

---

## Breadcrumbs
- **Category page**: Home (link) / intermediates (plain text) / leaf (plain text, highlighted)
- **Product page**: Home (link) / intermediates (plain text) / last category (link) / product slug
- Rule: only Home and the last/leaf node are clickable — intermediates are never linked (clicking them leads to empty pages)

---

## Next.js Gotchas

### `redirect()` in server actions
- Calling `redirect()` inside a server action throws `NEXT_REDIRECT` — code after it in the client never runs
- Solved by removing `redirect()` from `signIn`/`signOut` and handling navigation client-side

### Soft navigation refetches server components
- `router.push()` triggers background refetch of the current page's server components
- If session is cleared before navigation, `requireUser()` fails → forced redirect to `/signin`
- Fix: start navigation before clearing session

### Suspense boundary caching
- Without a `key` prop, Suspense reuses its component instance across URL param changes
- Next.js caches the result and doesn't re-fetch — `key={filter}` forces full remount

### `onAuthStateChange` event behavior
- `INITIAL_SESSION`: fires on every page load, with or without a session
- `SIGNED_IN`: only fires when the Supabase browser client performs the sign-in
- `SIGNED_OUT`: fires when the browser client clears the session
- Server-side sign-in (via server action) → `SIGNED_IN` never fires on the browser client
