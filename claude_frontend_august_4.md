# Try Before You Buy — Build Notes (August 4, 2026)

## What It Is

TBYB is a 4-step form that lets customers select a package of prescription sunglasses to try at home. They pay a deposit via Stripe, frames are shipped, and the deposit is either applied to a purchase or refunded minus a service fee.

---

## Flow

1. **Step 1 — Prescription**: OD and OS sphere, cylinder, axis. Axis is disabled until cylinder is selected and enabled only when cylinder is not "None".
2. **Step 2 — Fitting Questions**: Lens type, helmet size, hat size, nose bridge, sunglass fit, frame type.
3. **Step 3 — Contact Info**: Name (pre-populated from `user_metadata`), email, phone. Also accepts prescription and headshot file uploads (uploaded to Supabase Storage on file select, before submission).
4. **Step 4 — Review**: Displays all entered details. Submit calls `submitTBYB` → backend returns a Stripe checkout URL → `router.push(url)`.

After Stripe:
- **Cancel** → `/try-before-you-buy` (form state restored from localStorage)
- **Success** → `/` for now

---

## Packages

Seven packages seeded in `tbyb_packages`. Slugs follow `slugify()`:

```ts
str.toLowerCase().trim().replace(/[^a-z0-9]+/g, "-").replace(/^-+|-+$/g, "")
```

| Name | Slug | Price | Pairs | Brands |
|------|------|-------|-------|--------|
| BikerArmour | `bikerarmour` | $229 | 3–5 | BikerArmour |
| Wiley X | `wiley-x` | $249 | 3–5 | Wiley X |
| 7Eye | `7eye` | $249 | 3–5 | 7Eye |
| BikerArmour + Wiley X | `bikerarmour-wiley-x` | $279 | 5–8 | BikerArmour, Wiley X |
| BikerArmour + 7Eye | `bikerarmour-7eye` | $279 | 5–8 | BikerArmour, 7Eye |
| BikerArmour + 7Eye + Wiley X | `bikerarmour-7eye-wiley-x` | $329 | 7–10 | BikerArmour, 7Eye, Wiley X |
| 7Eye Ziena | `7eye-ziena` | $349 | 3 | 7Eye |

Images live in Supabase Storage: `bikershades/packages/<slug>.webp` (except 7Eye Ziena which reuses `7eye.webp`).

---

## API

### `submitTBYB(submission, successUrl, cancelUrl)`

```ts
export async function submitTBYB(submission: TBYBSubmission, successUrl: string, cancelUrl: string): Promise<CheckoutUrl>
```

Sends `{ brandSlug, submission, successUrl, cancelUrl }` to `/api/user/tbyb`. The backend:
1. Fetches the package by `packageId`
2. Builds a `form_hash` (SHA-256 of all fields + userId) for idempotency
3. Inserts into `tbyb_submissions` with status `"Unpaid"` (or reuses existing row on duplicate hash)
4. Creates a Stripe checkout session with `idempotencyKey: submissionId`
5. Returns `{ url: string }` — the Stripe checkout URL

### `getSubmissions()`

```ts
export async function getSubmissions(): Promise<TBYBSubmissionRecord[]>
```

POST `/api/user/submissions` — returns the user's submission history, newest first.

---

## Body Structure

Submission data is nested under `submission` to separate it from top-level config fields:

```json
{
  "brandSlug": "bikershades",
  "successUrl": "https://...",
  "cancelUrl": "https://...",
  "submission": {
    "packageId": "uuid",
    "odSphere": "-1.25",
    ...
  }
}
```

Null/optional fields are normalized to `"None"` in the client before calling `submitTBYB`. Never sent as `null`.

---

## Database

### `tbyb_packages`
Stores package definitions. `slug` is unique. `brands` is `text[]`.

### `tbyb_submissions`
Key columns beyond the form fields:
- `buying_preference` — maps to `sunglassFit` from the frontend
- `special_requests` — maps to `comments` from the frontend  
- `contact_name/email/phone` — maps to `name/email/phone`
- `form_hash text not null` — SHA-256 for idempotency. Unique index only where `status = 'Unpaid'` (allows resubmission after paying)
- `stripe_session_id`, `stripe_payment_intent` — populated by webhook
- `status` values: `Unpaid` → `Processing` → `Emailed` → `Curating` → `Shipped` → `Received`. `Refunded` set by webhook on full refund.

Package details are snapshotted onto the submission row (name, slug, price, image, pairs, brands) so historical records are preserved if packages change.

---

## localStorage Persistence

Form state is saved to `${brandSlug}:tbyb` after each step is validated. On mount, the client reads from localStorage and restores `vals`, `selectedPkg`, `step`, `rxUrl`, `photoUrl`. File objects (`rxFile`, `photoFile`) cannot be serialized — they're excluded, but their already-uploaded URLs are saved.

This means cancelling Stripe checkout returns the user to `/try-before-you-buy` with the full form pre-filled at step 4 (review), ready to resubmit.

`useEffect` is required for localStorage access because it's a browser API — server-side rendering doesn't have it.

---

## Null Normalization Convention

Optional fields are normalized to the string `"None"` rather than `null` before submission. This is done in the client (`TBYBClient.tsx`) not in `api.ts`, keeping the API layer clean. The DB stores `text not null` for all fields.

---

## Idempotency

If the user submits the same form twice (e.g. hits back after Stripe and resubmits), the backend detects the duplicate via `form_hash` and reuses the existing `Unpaid` row. The Stripe session is re-created with `idempotencyKey: submissionId`, returning the same session URL.

---

## Stripe Webhook

`checkout.session.completed` dispatches on `session.metadata.type`:
- `"order"` — creates order + order_items rows
- `"tbyb"` — updates submission status to `Processing`, stores session/payment IDs

`charge.refunded` — tries orders first, falls back to updating `tbyb_submissions` to `Refunded`.

---

## Concepts Learned

**`box-shadow: 0 0 0 1px`** — visual border that doesn't affect box model or layout, unlike CSS `border`.

**Slugify convention**: `str.toLowerCase().trim().replace(/[^a-z0-9]+/g, "-").replace(/^-+|-+$/g, "")` — any run of non-alphanumeric characters (spaces, `+`, etc.) becomes a single dash.

**Supabase refresh token rotation**: Access token (AT) expires → refresh token (RT) used to get new AT + new RT → old RT invalidated. Middleware is the only layer that can write new cookies back to the browser (server components silently fail). This is why a proxy/middleware is needed even when all auth logic is in server components.

**Suspense + streaming vs skeleton**: `revalidate` caches data but `<Suspense>` enables streaming — the page shell is sent immediately and suspended content arrives separately. Even with a cache hit, there's a brief skeleton flash. Removing Suspense blocks the full render but eliminates the flash.

**localStorage is synchronous** — `getItem`/`setItem` block and return immediately. `useEffect` is required not because it's async but because localStorage doesn't exist on the server during SSR.

**`setSubmitting(false)` only in catch** — on success, `router.push` navigates away and the component unmounts, so resetting state is pointless. Keeping the button in "Submitting…" during the redirect gives the user feedback. Only on error (when navigation never happens) do we reset so the user can retry.
