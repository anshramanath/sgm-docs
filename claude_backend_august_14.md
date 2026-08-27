# Rx Orders — Design & Implementation Notes (August 14, 2026)

## What Was Built

### Endpoints
- **`POST /api/user/rx-order`** — submits a prescription frame order, optionally using a TBYB deposit. Always goes through Stripe (minimum $0.50 charge so shipping is always collected).
- **`POST /api/user/deposit`** — returns available deposit for a TBYB submission identified by its short ID (last 8 chars of UUID).
- **`GET /api/public/prescriptions`** — returns prescription frames for a brand, sorted by name.
- **`POST /api/user/upload`** — updated to require a `folder` param so TBYB and rx uploads go to separate storage paths.

### DB Tables
- **`rx_orders`** — full snapshot order table. See 006_rx_orders.sql.
- **`tbyb_submissions`** — two new columns: `deposit_cents` (set by TBYB webhook) and `open_stripe_session_id` (tracks live rx checkout session).

### Webhook (`/api/webhooks/stripe`)
- **`tbyb` case** — sets `deposit_cents = package_price - 3000` (service fee always $30).
- **`rx-order` case** — sets order to `Processing`, updates `deposit_cents = depositLeftCents` on the TBYB submission, nulls `open_stripe_session_id`.
- **`charge.refunded` case** — sets `refunded_cents` on the submission; available deposit is computed dynamically as `deposit_cents - refunded_cents`.

---

## rx_orders Table Design

```sql
rx_orders (
  id, brand_slug, user_id,
  -- frame snapshot (so catalog changes don't affect historical orders)
  frame_name, frame_slug, frame_image_src, frame_price_cents, frame_color_slug,
  -- pricing snapshot
  deposit_used_cents, charge_cents,
  -- idempotency
  form_hash, status, stripe_session_id, stripe_payment_intent,
  -- prescription
  vision_type, od_sphere, od_cylinder, od_axis,
  os_sphere, os_cylinder, os_axis,
  pd_mode, pd, pd_left, pd_right,
  -- lens options
  lens_material, lens_color_category, lens_color,
  ar_coating, scratch_coating, mirror_coating,
  -- other
  comments, prescription_url, headshot_url,
  contact_name, contact_email, contact_phone,
  created_at, updated_at
)
```

**Statuses:** `Unpaid` → `Processing`

**Partial unique index:**
```sql
create unique index rx_orders_form_hash_unpaid
  on rx_orders (form_hash) where status = 'Unpaid';
```

---

## Deposit System

**Deposit set:** TBYB webhook fires → `deposit_cents = package_price - 3000`

**Available deposit (computed at read time):**
```
available = max(deposit_cents - refunded_cents, 0)
```

**Deposit consumed by rx order:**
```
deposit_used = min(available, frame_price)
deposit_left = deposit_cents - deposit_used   ← stored in Stripe metadata
charge        = max(frame_price - available, 50)  ← minimum $0.50
```

**Webhook update (atomic — direct set, no read needed):**
```
deposit_cents = depositLeftCents   (from metadata)
```

---

## Idempotency

### Form Hash
All rx_order fields are hashed (SHA-256) — the hash is derived by `JSON.stringify(orderRow)`, so it covers frame snapshot, pricing (including `deposit_used_cents` and `charge_cents`), full prescription, lens options, and contact info. `orderRow` is the single source of truth; the hash changes if any field changes.

### Insert + 23505 Handler
PostgREST upsert cannot target partial unique indexes (requires `ON CONFLICT (col) WHERE predicate` which PostgREST doesn't support). So we use:
```
insert → if 23505 (unique_violation) → look up existing row by form_hash + status = 'Unpaid'
```
This reuses the existing `orderId`, and the Stripe idempotency key (`orderId`) returns the same session.

### Stripe Idempotency Key
`orderId` is passed as the Stripe idempotency key. Same key + same params → cached session. Same key + different params → Stripe 400. Since params are tied to the hash, both change together.

---

## Double-Spend Prevention

`open_stripe_session_id` on `tbyb_submissions` tracks the one live rx checkout session per TBYB submission. The webhook atomically updates `deposit_cents` AND nulls `open_stripe_session_id` in a single write — so reading null session ID always means the deposit is current.

**On new rx-order request (after session creation):**
1. If `session.id === open_stripe_session_id` → same session, idempotent — return URL immediately
2. If different session and `open_stripe_session_id` is set → expire it → null in DB
   - If expire fails with `complete` → 500 (webhook hasn't fired yet — wait)
3. IS NULL claim: `UPDATE tbyb_submissions SET open_stripe_session_id = session.id WHERE id = X AND open_stripe_session_id IS NULL`
   - 0 rows → 409 (simultaneous race lost)

**Why complete → 500 is correct:**
`open_stripe_session_id` being non-null and complete means payment happened but the webhook hasn't written the new deposit yet. Blocking with 500 forces the caller to wait until the webhook fires and nulls the session ID, at which point the deposit is current and a new request can proceed safely.

**Why IS NULL on the write:**
Two simultaneous requests with different form data can both read `open_stripe_session_id = null` and both create sessions. The IS NULL update is atomic — first write wins, second gets 0 rows and 409s. The losing session dangles in Stripe and expires on its own.

**Webhook:** Atomically sets `deposit_cents = depositLeftCents` (from metadata) and `open_stripe_session_id = null` in one write.

**Why deposit overwrites are safe:** Two completed webhooks for the same TBYB submission cannot race. The one-active-session policy guarantees it — IS NULL write ensures only one session is ever tracked, expiry kills the old session before the new one is claimed, and complete → 500 blocks any new session while payment is pending webhook. A session that loses the IS NULL race is never returned to the user, so it can never be paid. There is no code path where two different completed webhooks fire for the same submission.

---

## Design Progression

### 1. Hash → orderRow derivation
Initially the hash was a manually maintained parallel object — the same fields listed twice. Realized the hash should be derived directly from `orderRow` so there's a single source of truth: build `orderRow` first, then `JSON.stringify(orderRow)` into the hash. Any field change automatically propagates.

### 2. frame_id removed
`rx_orders` originally had a `frame_id` FK. Removed because the full frame snapshot (`frame_name`, `frame_slug`, `frame_image_src`, `frame_price_cents`, `frame_color_slug`) makes it redundant for display and historical purposes. `frame_id` would only add value for joining back to the live catalog, which isn't needed.

### 3. Expiry moved after session creation
Originally the flow expired `open_stripe_session_id` before creating the Stripe session. This meant every retry would expire and recreate — making the hash, `orderId`, and Stripe idempotency key pure decoration (Stripe had to create a fresh session every time).

By moving expiry after session creation, the same-session check becomes possible:
```
if (session.id === tbybSub.open_stripe_session_id) → skip everything, return URL
```
Now the hash earns its keep: same inputs → same `orderId` → Stripe returns the same session → early return with no side effects.

### 4. True idempotency — what was missing

**Problem 1 — Simultaneous different requests both reading null:** Two requests with different form data both read `open_stripe_session_id = null`, both create different sessions, last write wins. Both sessions stay live in Stripe.

Fix: null the DB after expiry, then use IS NULL as an atomic claim on write. First write wins, second gets 0 rows → 409.

**Problem 2 — IS NULL breaks same-session retry:** The IS NULL write runs even when `session.id === open_stripe_session_id` (idempotent retry). DB value is the existing session ID, not null → 0 rows → 409 on every retry.

Fix: guard the IS NULL write with `if (session.id !== tbybSub.open_stripe_session_id)`. Same session → skip write entirely.

**Problem 3 — Fresh deposit re-read (considered and removed):** Initially added a re-read of `deposit_cents` after expiry to catch webhook races. Removed because the webhook atomically writes `deposit_cents` and `open_stripe_session_id` together — if you read null session ID, the deposit is already current. The only remaining race (webhook fires after read but before expiry, with old session set) is caught by the complete → 500 path, since the session would be complete when we try to expire it.

### 5. Final flow (TBYB path)
```
1. Read tbyb submission — validate depositCents matches DB
2. Build orderRow → hash → insert (23505 → look up) → orderId
3. Create Stripe session (idempotency key = orderId)
4. If session.id === open_stripe_session_id → return URL (idempotent short-circuit)
5. If open_stripe_session_id is set and different → expire → null in DB
   (complete → 500, other error → 500)
6. IS NULL write to claim session slot → 409 if race lost
7. Return URL
```

---

## Things Learned

### PostgREST / Supabase

| Pattern | Works? | Notes |
|---|---|---|
| `upsert` no `onConflict` | PK only | Conflicts on any other unique index surface as 23505 |
| `upsert` with `onConflict: "col"` | Full unique index only | Partial indexes not supported |
| `upsert` with `onConflict: "col_a,col_b"` | Composite full index | Same restriction |
| Partial unique index | Insert + 23505 handler | Only way via PostgREST |
| `.single()` | Returns row or PGRST116 | PGRST116 = 0 or >1 rows |
| `.select()` on insert | Only returns on success | Null on conflict/error |
| Expression in update value | Not supported | Need RPC for `col = col - x` |
| Atomic read-modify-write | Not supported | Use RPC or pre-compute new value and pass directly |

### Stripe
- Idempotency key is tied to exact request params — same key + different params → 400
- Cannot expire a completed/already-expired session → `invalid_request_error`
- Minimum charge in USD: **$0.50** (50 cents)
- Metadata values are always strings; passing `null` or `undefined` causes Stripe to drop the key entirely
- TBYB metadata keys (`tbybSubmissionId`, `depositLeftCents`) are conditionally spread — absent for non-TBYB orders, so `session.metadata.tbybSubmissionId` is `undefined` on the webhook side
- `"0"` in metadata is a non-empty string and is truthy — safe to gate on `if (tbybSubmissionId && depositLeftCents)` even when deposit left is zero

### General
- `JSON.stringify` serializes `null` as `"null"` — safe to hash objects with null values
- Postgres `numeric` columns return as strings from supabase-js — wrap with `Number()`
- `.filter("id::text", "ilike", ...)` doesn't work for UUID columns — pull all rows and filter in JS
- UUID last-8-char short ID: `id.slice(-8).toUpperCase()`
- 23505 = Postgres unique_violation error code
- PGRST116 = PostgREST error for `.single()` returning wrong number of rows
- Atomically setting a value: pre-compute the final value and pass it directly (avoids read-modify-write); for `deposit_cents`, pass `depositLeftCents` in Stripe metadata so the webhook just does `SET deposit_cents = depositLeftCents`

---

## SQL to Run (006_rx_orders.sql)

Run the full contents of `src/lib/db/006_rx_orders.sql` in the Supabase SQL editor. This creates the `rx_orders` table, the partial unique index, RLS policy, and grants.
