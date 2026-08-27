# The Auth Flash Problem (August 4, 2026)

## The Problem

On every page load, logged-in users would briefly see "Sign In" before it flipped to the profile icon. A flash — small but visually jarring.

## Why It Happened

The navbar has two sources of truth:

1. **`isSignedIn`** — passed from the server component. Accurate from the start because it reads the session cookie during SSR.
2. **`loggedIn`** — from `AuthProvider`, a client-side context. Starts as `false`, then a `useEffect` runs `getUser()` and sets it to `true` if the user is authenticated.

`HeaderIcons` used both:

```ts
const isActuallySignedIn = isSignedIn && loggedIn;
```

The problem: even though `isSignedIn` was `true` from the server, `loggedIn` started as `false` — so `isActuallySignedIn` was `false` on first render. The `useEffect` hadn't run yet. For one frame (or more), "Sign In" rendered. Then the effect ran, `loggedIn` became `true`, and the icon swapped in.

The server knew the user was logged in. The client didn't — yet.

## Why `loggedIn` Existed At All

`loggedIn` isn't redundant. It's needed for sign-out: when the user signs out, `isSignedIn` stays `true` (it was baked in at SSR time and won't change until the next page load). `setLoggedIn(false)` is the only way to immediately update the navbar without a full reload. The client-side state is the escape hatch for mutations that happen after the page renders.

## The Fix

Change `loggedIn`'s initial value from `false` to `null`, representing "not yet determined":

```ts
// Before
const [loggedIn, setLoggedIn] = useState(false);

// After
const [loggedIn, setLoggedIn] = useState<boolean | null>(null);
```

Then in `HeaderIcons`, use `?? true` to fall back to trusting the server when `loggedIn` is still undetermined:

```ts
// Before
const isActuallySignedIn = isSignedIn && loggedIn;

// After
const isActuallySignedIn = isSignedIn && (loggedIn ?? true);
```

When `loggedIn` is `null` (effect hasn't run yet), `?? true` kicks in and defers to `isSignedIn`. The server already knows — trust it. Once the effect resolves and sets `loggedIn = true`, nothing changes visually. No flash.

## Why Nothing Else Broke

`null` is falsy in JavaScript, so every other consumer of `loggedIn` — CartProvider, BookmarkProvider — behaves identically to the old `false` default. They use `if (loggedIn)` to decide whether to sync with the DB, so `null` just means "not yet, wait" — same behavior as before.

## The Insight

Three states are needed, not two:

| State | Meaning |
|-------|---------|
| `null` | Undetermined — effect hasn't run |
| `true` | Confirmed logged in |
| `false` | Confirmed logged out (or signed out client-side) |

Using `false` as the initial state collapsed "undetermined" and "logged out" into the same value. Separating them with `null` let the navbar make the right call immediately using the server's knowledge, rather than waiting for the client to catch up.
