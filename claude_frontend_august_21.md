# Session Notes (August 21, 2026)

## Rx Route Files

### `src/app/(shop)/rx/page.tsx`
- Passes `email` and `name` as plain `string` (using `?? ""`) so client components never receive `undefined` — no `?? ""` needed inside clients.

### `src/app/(shop)/rx/TBYBClient.tsx`
- Props: `email: string; name: string`
- State starts as `{ ...INIT }` (empty name/email); `useEffect` restores from localStorage or falls back to props.
- Stored case: `{ ...prev, ...(savedVals ?? {}), name: savedVals?.name || name, email: savedVals?.email || email }` — `||` catches empty string (INIT always writes `""`) and falls back to the account value.
- No-stored case: `setVals(prev => ({ ...prev, name, email }))`.
- Email validated with regex before submit: `/^[^\s@]+@[^\s@]+\.[^\s@]+$/`.
- No `!email || !name` auth guard — auth is enforced at the API layer (401).
- All text inputs/textareas: `text-base` (16px) to prevent iOS zoom. Dropdown trigger buttons stay `text-[15px]` (buttons don't trigger zoom).

### `src/app/(shop)/rx/PrescriptionFramesClient.tsx`
- Same prop/state pattern as TBYBClient.
- `mergedVals` uses `{ ...vals, ... }` as base (not `{ ...INIT, ... }`) to preserve any changes made before the effect runs.
- Same 16px input rule.

### TBYB description (final)
> "Pay a deposit and we'll curate frames that fit you and match your preferences. Try them at home, send them all back, and place an Rx Frame order for the one you want. The $30 service fee is always deducted — the rest applies toward your frame or is refunded."

---

## Footer Pages

### FAQ (`src/app/(shop)/(footer)/faq/page.tsx`)
- Status section title: "What do these order statuses mean?"
- All references updated: "Rx order" → "Rx Frame order".
- Emailed status: "Check your inbox — we need a reply to move forward."
- "Try Before You Buy (TBYB)" and "Rx Frames" are linked to `/rx` and `/rx?tab=frames` respectively.
- Rx Frames defined as prescription glasses (frame + lenses built to your prescription) — not compared to an order.
- Account requirement: no account needed to start the forms; account required to checkout.
- Deposit note: "$30 is always deducted — the remaining balance applies toward your Rx Frame order or is refunded once we receive the frames back."

### Privacy (`src/app/(shop)/(footer)/privacy/page.tsx`)
- Removed "order history, and order status" from collected data list (unnecessary).
- Section heading: "Cookies & Local Storage" (both words capitalised).
- localStorage description: saves cart, bookmarks, and in-progress TBYB/Rx Frame details; cart and bookmarks also sync to DB when signed in.
- Last updated: August 20, 2026.

### Returns (`src/app/(shop)/(footer)/returns/page.tsx`)
- Partial refund phrasing: "covering the item price".

### Shipping (`src/app/(shop)/(footer)/shipping/page.tsx`)
- "Shipping and tax are included in the price of every order."
- "We currently only ship within the United States."

---

## iOS Zoom Fix

Browsers zoom in on inputs with `font-size < 16px`. Fixed by setting all `<input>` and `<textarea>` elements in the Rx forms to `text-base` (16px). Dropdown trigger `<button>` elements are exempt — buttons don't trigger zoom. Sign-in, reset-password, and search inputs were already at 16px.

---

## SQL Schema

Five migration files (`001`–`005`) reviewed and confirmed in sync with `initial_schema.sql` and `drop_schema.sql`.

Key intentional decisions:
- `products.total_sales int` — nullable. Simple products (empty attributes array) are initialised to `0` by the backend. Variable products (non-empty attributes) have `null` because only their variations are sellable; variations track sales via `variations.total_sales int not null`.
- `categories.view_count int default null` — nullable, incremented via `increment_category_view()` function using `coalesce`.
- RLS enabled on all user-facing tables; no `anon` grants because public catalog data is fetched via a service-role admin endpoint, not the client.

---

## ProductDetail — Attribute Display

`src/app/(shop)/product/[slug]/ProductDetail.tsx`

Attribute names are capitalised for display only (not stored in DB):
```tsx
attrName.charAt(0).toUpperCase() + attrName.slice(1)
```

---

## Order Success Redirect

`src/app/(bare)/order/success/page.tsx` — redirects to `/account` after the countdown (was `/`).

---

## Account Page — "Refunded" Label

`src/app/(bare)/account/page.tsx` — always shows "Refunded" regardless of amount. "Partially Refunded" was removed.

---

## Pending: submitTBYB Server Component Error

`redirect()` from `next/navigation` is server-only. It's currently called inside `submitTBYB`, `submitRxOrder`, `getDeposit`, and `uploadFile` in `src/lib/api.ts` on 401/default cases. When called from a client component this throws `NEXT_REDIRECT`, which surfaces as "An error occurred in the Server Components render."

**Fix needed:** Replace `redirect("/sign-in")` in those four client-callable functions with `throw new Error("Unauthorized")` and handle it in the client (e.g. `router.push("/sign-in")`).

Functions that call `redirect()` on 401 but are server-only (`getOrders`, `getSubmissions`, `getRxOrders`) are fine as-is.

---

## Sign-in Flash in Production (Non-issue)

A white-screen flash to `/sign-in` was observed on `/` in production but not in a private tab. Determined to be a Next.js router cache artefact from switching between dev and prod environments — not something a real user would encounter.
