# Sunglass Server — Build Notes (August 4, 2026)

## Architecture Overview

Multi-brand e-commerce platform built on Next.js (App Router), Supabase, and Stripe.

**Three Supabase client types:**
- `createAdminClient` — service role, bypasses RLS. Used in API routes that need trusted DB access.
- `createUserClient` — validates the `Authorization: Bearer <token>` header, returns `{ supabase, user }`. Used in authenticated API routes.
- `createServerClient` — reads cookies, handles token refresh. Used in SSR pages and the admin dashboard.

**Auth pattern by surface:**
- Public API routes → `createAdminClient` (no auth needed, brand-scoped queries)
- User API routes → `createUserClient` (validates JWT, queries scoped to user_id via RLS)
- Admin dashboard → `createServerClient` + admin check (reads from `admins` table)

Cookie refresh happens inside `createServerClient` — every call to `getUser()` refreshes the session if the access token is expired and a refresh token is available. No separate middleware or proxy needed because the admin dashboard is server-rendered and can write `Set-Cookie` headers directly.

---

## TBYB — Try Before You Buy

### What changed

Replaced a simple form-to-DB submission with a full Stripe checkout flow. The form data is saved before payment and the submission status is updated by webhook after payment completes.

### Full flow

1. Frontend submits form → `POST /api/user/tbyb`
2. Backend fetches package details from `tbyb_packages` (snapshot, not just ID)
3. Computes SHA-256 form hash from all fields + package details + userId
4. Inserts row into `tbyb_submissions` with `status = "Unpaid"`
5. Creates Stripe checkout session, returns `{ url }`
6. Frontend redirects user to Stripe
7. `checkout.session.completed` webhook fires → updates submission to `"Processing"`, stores `stripe_session_id`, `stripe_payment_intent`, and `shipping_address`
8. On full refund: `charge.refunded` webhook fires → updates submission to `"Refunded"`

### Duplicate prevention

**Problem:** Cancelling Stripe and resubmitting the same form would insert a new row every time, accumulating stale Unpaid rows.

**Solution:** SHA-256 hash of the entire form state + package snapshot + userId, stored as `form_hash`. A partial unique index enforces uniqueness only while the row is Unpaid:

```sql
create unique index tbyb_submissions_form_hash_unpaid
  on tbyb_submissions (form_hash) where status = 'Unpaid';
```

**Race condition:** If two concurrent requests insert the same hash, one succeeds and the other gets a `23505` (unique_violation) error from Postgres. The loser looks up the existing Unpaid row by hash and reuses its `submissionId`. Both requests then call Stripe with the same idempotency key (`submissionId`) and get back the same checkout URL.

**Why `submissionId` as idempotency key, not the hash:** The hash is an intermediate artifact. The submission ID is the durable identity — "the hash gets exchanged for an idempotent ID."

**Resubmission after payment:** The partial index only covers `status = 'Unpaid'`. Once paid (status becomes `"Processing"`), the constraint no longer applies to that row, so the user can submit the same form again and a new Unpaid row is created.

### Form hash contents

```
userId, packageName, packagePriceCents, packagePairsMin, packagePairsMax,
packageBrands, packageImageSrc,
odSphere, odCylinder, odAxis, osSphere, osCylinder, osAxis,
lensType, helmetSize, hatSize, noseBridge, sunglassFit, frameType,
comments, prescriptionUrl, headshotUrl, name, email, phone
```

Package details (not just ID) are included so that a price or name change produces a different hash — the user would get a new session at the new price, not a reuse of the old one.

### Status model

| Status | Set by | When |
|--------|--------|------|
| `Unpaid` | System | On form submission, before payment |
| `Processing` | System (webhook) | After successful payment |
| `Emailed` | Admin | After reaching out to customer |
| `Curating` | Admin | While selecting glasses |
| `Shipped` | Admin | After package ships |
| `Received` | Admin | After package is returned |
| `Refunded` | System (webhook) | After full Stripe refund |

`Unpaid` and `Refunded` are terminal states — the admin dashboard shows a fixed non-editable status box for these instead of the dropdown.

### DB schema additions

```sql
form_hash             text not null,
stripe_session_id     text,           -- null until paid
stripe_payment_intent text,           -- null until paid
shipping_address      jsonb,          -- null until paid; stored by webhook
```

### Webhook dispatch

The webhook inner-switches on `session.metadata.type`:

- `"tbyb"` → update submission status to `"Processing"`, store Stripe IDs and shipping address
- `"order"` → create `orders` + `order_items` rows (idempotent via `stripe_session_id`)
- unknown → return 400

Both `POST /api/user/tbyb` and `POST /api/user/checkout` set `metadata.type` so the webhook knows what to do.

### Refund routing

`charge.refunded` has no brand context (no `metadata` on a charge event). Matching is done by `stripe_payment_intent`, which is globally unique in Stripe.

Orders are tried first via `.select("id")`. If `updatedOrders.length === 0`, the payment intent belongs to a TBYB submission instead, and that row is updated to `"Refunded"`.

---

## Other Changes

### `POST /api/user/tbyb`
- Stripe collects shipping address at checkout (`shipping_address_collection: { allowed_countries: ["US"] }`)
- Stripe SDK call wrapped in try/catch

### `POST /api/user/checkout`
- Added `type: "order"` to `metadata` so the webhook can dispatch correctly
- Removed `.slice(0, 32)` from idempotency key hash — full 64-char hex is safer
- Stripe SDK call wrapped in try/catch

### `POST /api/user/submissions`
- Added `shipping_address` to select and response map — `null` until payment completes

### `GET /api/public/sale`
- Added `sale` to the select query and response map — components depend on this field for the sale label

---

## Patterns and Conventions

**Supabase error vs null data:**
```ts
if (pkgError) return err("Failed to fetch package", 500);  // DB failure
if (!pkg) return err("Package not found", 404);             // zero rows
```
Always distinguish — `.single()` returns an error for zero rows, so `pkgError` and `!pkg` are separate conditions.

**Stripe SDK calls always need try/catch:**
```ts
let session;
try {
  session = await stripe.checkout.sessions.create({ ... });
} catch {
  return err("Failed to create checkout session", 500);
}
```
Stripe throws; it doesn't return `{ data, error }`.

**Brand scoping:** Every DB query that has access to `brandSlug` includes `.eq("brand_slug", brandSlug)`. The only exception is `charge.refunded` — no brand context is available in that event, so the match is purely on `stripe_payment_intent`.
