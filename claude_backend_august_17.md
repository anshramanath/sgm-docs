# Rx Orders (August 17, 2026)

Full prescription lens order flow for BikerShades. Customers select a prescription frame, configure lens options, optionally apply a TBYB deposit, and pay via Stripe checkout.

---

## Schema — `rx_orders`

```sql
create table rx_orders (
  id                    uuid        primary key default gen_random_uuid(),
  brand_slug            text        not null references brands(slug) on delete cascade,
  user_id               uuid        references auth.users(id) on delete set null,

  -- Frame snapshot (locked at order time)
  frame_name            text        not null,
  frame_slug            text        not null,
  frame_image_src       text        not null,
  frame_price_cents     int         not null,
  frame_color           text        not null,   -- display name resolved server-side from colors JSONB

  -- Pricing
  total_price_cents     int         not null,   -- frame + all addon costs (true order value)
  deposit_used_cents    int,                    -- null for non-TBYB; capped at total_price_cents
  stripe_charge_cents   int         not null,   -- what Stripe charged (min $0.50 floor)
  refunded_cents        int,                    -- null until refunded; cumulative cents

  -- Stripe
  stripe_session_id     text,
  stripe_payment_intent text,
  shipping_address      jsonb,                  -- null until webhook fires on payment

  -- Idempotency
  form_hash             text        not null,

  -- Status
  status                text        not null,   -- Unpaid | Processing | Emailed | Shipped | Refunded

  -- Prescription
  vision_type           text        not null,
  od_sphere             text        not null,
  od_cylinder           text        not null,
  od_axis               text        not null,
  os_sphere             text        not null,
  os_cylinder           text        not null,
  os_axis               text        not null,
  pd_mode               text        not null,   -- "Single" | "Dual"
  pd                    text        not null,
  pd_left               text        not null,
  pd_right              text        not null,

  -- Lens options
  lens_material         text        not null,
  lens_color_category   text        not null,
  lens_color            text        not null,
  ar_coating            text        not null,
  scratch_coating       text        not null,
  mirror_coating        text        not null,

  -- Fulfillment
  carrier               text,
  tracking_number       text,

  -- Contact (snapshot)
  comments              text        not null,
  prescription_url      text        not null,
  headshot_url          text        not null,
  contact_name          text        not null,
  contact_email         text        not null,
  contact_phone         text        not null,

  created_at            timestamptz not null default now(),
  updated_at            timestamptz not null default now()
);

create unique index rx_orders_form_hash_unpaid
  on rx_orders (form_hash) where status = 'Unpaid';
```

**Status values:** `Unpaid` → `Processing` → `Emailed` → `Shipped` → `Refunded`

- `Unpaid`: set on insert before payment
- `Processing`: set by webhook on successful payment
- `Emailed`, `Shipped`: set by admin via status dropdown
- `Refunded`: set by webhook on full refund (partial refund updates `refunded_cents` only, status unchanged)

**Refund detection:**
- Partial: `refunded_cents !== null && status !== "Refunded"`
- Full: `status === "Refunded"`

---

## Pricing

All lens option prices are computed server-side from hardcoded dicts. The client sends option label strings; the server looks them up. Unknown keys → `400`. This prevents client-side price manipulation.

```
total_price_cents     = frame_price_cents + sum(all addon prices)
stripe_charge_cents   = max(total_price_cents - deposit_used_cents, 50)
```

`stripe_charge_cents` ≠ true order value due to the $0.50 Stripe minimum. True amount owed = `total_price_cents - deposit_used_cents`.

**Price dicts:** `VISION_TYPE_PRICES` (5), `LENS_MATERIAL_PRICES` (2), `LENS_COLOR_PRICES` (24), `AR_COATING_PRICES` (4), `SCRATCH_COAT_PRICES` (2), `MIRROR_COAT_PRICES` (17).

---

## Frame Color Resolution

Client sends `frameColorSlug`. Server selects `colors` from `prescription_frames`, finds the matching entry by slug, stores `frameColor.option` (display name) in the snapshot. Unknown slug → `400`. DB never stores client-supplied slug values.

---

## TBYB Deposit

When `tbybSubmissionId` is provided:

1. Fetch matching submission (by last 8 chars of UUID, case-insensitive).
2. Compute `available = max(deposit_cents - refunded_cents, 0)`.
3. Validate `submission.depositCents === available` — if stale, return `422` with fresh amount.
4. `deposit_used_cents = min(available, total_price_cents)` — capped so deposit never exceeds order value.
5. `deposit_left_cents = deposit_cents - deposit_used_cents` — passed through Stripe metadata.
6. On webhook: `tbyb_submissions.deposit_cents` set to `deposit_left_cents`, `open_stripe_session_id` cleared.

---

## Idempotency

`form_hash = SHA256(JSON.stringify(orderRow))` over the full snapshot (excluding `form_hash` and `status`). Partial unique index on `(form_hash) where status = 'Unpaid'` enforces one pending order per unique input set. On `23505` conflict, returns existing row ID and creates a new Stripe session with it as the idempotency key.

---

## Stripe Session

- `idempotencyKey`: `rx_orders.id` UUID — same order always reuses the same Stripe session
- `metadata`: `{ brandSlug, type: "rx-order", rxOrderId, tbybSubmissionId?, depositLeftCents? }` — TBYB keys absent (not `"null"`) for non-TBYB via conditional spread
- `customer_email`: pre-filled from `submission.email`
- Collects: shipping address, billing address, phone

---

## TBYB Session Race Prevention

Before returning the Stripe URL:

1. If `open_stripe_session_id` matches the new session → reuse, skip.
2. If different → expire the old session (ignore `invalid_request_error` for already-expired; error if completed).
3. Null out `open_stripe_session_id` on the submission.
4. Claim with `update ... is("open_stripe_session_id", null)` — if `claimed.length === 0`, race lost → `409`.

---

## Webhook

### `checkout.session.completed` — type `"rx-order"`

1. Update `rx_orders`: `status = "Processing"`, store `stripe_session_id`, `stripe_payment_intent`, `shipping_address`.
2. If `tbybSubmissionId && depositLeftCents` in metadata: update `tbyb_submissions` — set `deposit_cents = parseInt(depositLeftCents, 10)`, clear `open_stripe_session_id`.

### `charge.refunded`

Refund chain — tries each table in order, stops at the first match:

1. `orders` — updates `refunded_cents`; sets `status = "refunded"` on full refund only.
2. `rx_orders` — updates `refunded_cents`; sets `status = "Refunded"` on full refund only.
3. `tbyb_submissions` — updates `refunded_cents` and sets `status = "Refunded"` unconditionally.

---

## Admin Utilities — `src/lib/admin/rx-orders.ts`

| Function | Description |
|---|---|
| `getRxOrders(brandSlug)` | Fetches all rx orders newest first; maps snake_case → camelCase |
| `updateRxStatus(id, status)` | Sets status via dropdown; does not touch fulfillment |
| `saveRxFulfillment(id, carrier, trackingNumber)` | Saves carrier + tracking only; does not set status |
| `undoRxFulfillment(id)` | Clears carrier + tracking only; does not reset status |

---

## Admin Page — `src/app/admin/[brandSlug]/rx-orders/`

### Table columns
`Order · Placed · Customer · Total · Status · chevron`

Grid: `1.2fr 1.4fr 1.4fr 0.9fr 1.1fr 32px`

### Filters
All · Processing · Emailed · Shipped · Refunded · Partially Refunded

Partially Refunded matches `refunded_cents !== null && status !== "Refunded"`. Unpaid orders are always hidden.

### Status colors
- Processing, Emailed, Refunded: brand accent
- Shipped: black
- Unpaid: grey (filtered out, never shown)

### Expanded panel sections

**Row 1 — 3 columns:**
- Contact (name, email, phone) + Payment intent below
- Frame (name, color, total · paid, deposit if TBYB) + Refund label below (accent)
- Status dropdown + Save · Fulfillment (carrier dropdown + tracking + Save/Undo)

**Row 2 — 2 columns:**
- Prescription grid (OD/OS × Sphere/Cylinder/Axis)
- Vision type + Comments

**Row 3 — 4-column grid (single unified grid for row alignment):**
- PD · value · AR coating · value
- Lens material · value · Scratch coating · value
- Lens colour · value · Mirror coating · value

**Row 4 — 3 columns:**
- Shipping address (null → "None")
- Prescription upload (link or "None")
- Headshot photo (link or "None")

### Status dropdown
Options: Processing, Emailed, Shipped. Locked when Refunded.

### Fulfillment
Matches TBYB pattern exactly:
- Carrier + tracking only become interactive when `status === "Shipped"` and not yet saved
- Saving writes carrier + tracking only (does not auto-set status)
- Undo clears carrier + tracking only (does not reset status)
- Disabled entirely when Refunded

### PD display
`pd_mode === "Dual"` → `pdLeft / pdRight`. Otherwise → `pd`.

### Phone / address
`contactPhone === "None"` → display "None". `shippingAddress === null` → display "None".

---

## Sidebar

Rx Orders appears below TBYB in the BikerShades-only sidebar block. Both items are conditionally rendered behind `brandSlug === "bikershades"`.

---

## Files

| File | Purpose |
|---|---|
| `src/app/api/user/rx-order/route.ts` | POST endpoint |
| `src/app/api/webhooks/stripe/route.ts` | Webhook handler |
| `src/lib/admin/rx-orders.ts` | Admin server actions |
| `src/lib/types.ts` | `RxOrder` type |
| `src/app/admin/[brandSlug]/rx-orders/page.tsx` | Admin page (server component) |
| `src/app/admin/[brandSlug]/rx-orders/rx-orders-table.tsx` | Admin table (client component) |
| `src/components/sidebar.tsx` | Sidebar nav entry |
| `src/lib/db/005_prescription_frames.sql` | Migration |
| `src/lib/db/initial_schema.sql` | Full schema reference |
| `API.md` | API documentation |
