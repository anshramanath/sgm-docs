# Rx Orders (August 16, 2026)

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
  pd_mode               text        not null,
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
- `Emailed`, `Shipped`: set by admin
- `Refunded`: set by webhook on full refund (partial refund updates `refunded_cents` only)

**Partial vs full refund detection:**
- Partial: `refunded_cents !== null && status !== "Refunded"`
- Full: `status === "Refunded"`

---

## Pricing

All lens option prices are computed server-side from hardcoded dicts. The client sends option label strings; the server looks them up in the dicts. Unknown keys → `400`. This prevents client-side price manipulation.

```
total_price_cents = frame_price_cents + sum(all addon prices)
stripe_charge_cents = max(total_price_cents - deposit_used_cents, 50)
```

`stripe_charge_cents` is not the true order value because of the $0.50 Stripe minimum. True amount owed = `total_price_cents - deposit_used_cents`.

**Price dicts:** `VISION_TYPE_PRICES` (5), `LENS_MATERIAL_PRICES` (2), `LENS_COLOR_PRICES` (24), `AR_COATING_PRICES` (4), `SCRATCH_COAT_PRICES` (2), `MIRROR_COAT_PRICES` (17).

---

## Frame Color Resolution

The client sends `frameColorSlug`. The server selects `colors` from `prescription_frames`, finds the matching entry by slug, and stores `frameColor.option` (the display name) in the snapshot. Unknown slug → `400`. The DB never stores client-supplied slug values.

---

## TBYB Deposit

When `tbybSubmissionId` is provided:

1. Fetch the matching submission (by last 8 chars of UUID, case-insensitive).
2. Compute `available = max(deposit_cents - refunded_cents, 0)`.
3. Validate `submission.depositCents === available` — if stale, return `422` with fresh amount.
4. `deposit_used_cents = min(available, total_price_cents)` — capped so deposit never exceeds order value.
5. `deposit_left_cents = deposit_cents - deposit_used_cents` — passed through Stripe metadata.
6. On webhook: `tbyb_submissions.deposit_cents` is set to `deposit_left_cents` and `open_stripe_session_id` is cleared.

---

## Idempotency

`form_hash = SHA256(JSON.stringify(orderRow))` where `orderRow` is the full snapshot (excluding `form_hash` and `status`). A partial unique index on `(form_hash) where status = 'Unpaid'` enforces one pending order per unique set of inputs. On `23505` conflict, the existing `Unpaid` row's ID is returned and a new Stripe session is created with it as the idempotency key.

---

## Stripe Session

- `idempotencyKey`: the `rx_orders.id` UUID — same order always reuses the same Stripe session
- `metadata`: `{ brandSlug, type: "rx-order", rxOrderId, tbybSubmissionId?, depositLeftCents? }` — TBYB keys are absent (not `"null"`) for non-TBYB orders via conditional spread
- `customer_email`: pre-filled from `submission.email`
- Collects: shipping address, billing address, phone

---

## TBYB Session Race Prevention

Before returning the Stripe URL, if the submission already has an `open_stripe_session_id`:

1. If it matches the new session ID → reuse, skip (same session).
2. If different → expire the old session (ignore `invalid_request_error` for already-expired; error if it completed).
3. Null out `open_stripe_session_id` on the submission.
4. Claim with `update ... is("open_stripe_session_id", null)` — if `claimed.length === 0`, another request won the race → `409`.

---

## Webhook

### `checkout.session.completed` — type `"rx-order"`

1. Update `rx_orders`: `status = "Processing"`, store `stripe_session_id`, `stripe_payment_intent`, `shipping_address`.
2. If `tbybSubmissionId && depositLeftCents` in metadata: update `tbyb_submissions` — set `deposit_cents = parseInt(depositLeftCents, 10)`, clear `open_stripe_session_id`.

### `charge.refunded`

Refund chain — tries each table in order, stops at the first match:

1. `orders` — updates `refunded_cents`; sets `status = "refunded"` on full refund only.
2. `rx_orders` — updates `refunded_cents`; sets `status = "Refunded"` on full refund only.
3. `tbyb_submissions` — updates `refunded_cents` and sets `status = "Refunded"` on any refund.

---

## Admin

### `src/lib/admin/rx-orders.ts`

| Function | Description |
|---|---|
| `getRxOrders(brandSlug)` | Fetches all rx orders for a brand, newest first |
| `saveRxFulfillment(id, carrier, trackingNumber)` | Sets carrier, tracking, status → `Shipped` |
| `undoRxFulfillment(id)` | Clears carrier, tracking, status → `Processing` |

### Sidebar

Rx Orders appears below TBYB in the BikerShades-only sidebar section.

### Page filters (planned)

Same as Orders minus Veeqo error, plus `Emailed`: Processing, Emailed, Shipped, Refunded.

---

## Files

| File | Purpose |
|---|---|
| `src/app/api/user/rx-order/route.ts` | POST endpoint |
| `src/app/api/webhooks/stripe/route.ts` | Webhook handler |
| `src/lib/admin/rx-orders.ts` | Admin server actions |
| `src/lib/types.ts` | `RxOrder` type |
| `src/app/admin/[brandSlug]/rx-orders/page.tsx` | Admin page (stub) |
| `src/components/sidebar.tsx` | Sidebar nav entry |
| `src/lib/db/005_prescription_frames.sql` | Migration |
| `src/lib/db/initial_schema.sql` | Full schema reference |
