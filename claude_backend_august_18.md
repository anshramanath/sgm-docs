# Rx Orders (August 18, 2026)

## Overview

Rx Orders are prescription lens orders tied to a specific frame. The user selects a frame, chooses lens options, optionally applies a TBYB deposit, and pays via Stripe Checkout. The admin reviews the prescription details and ships the lenses.

---

## Schema (`rx_orders`)

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | PK |
| `brand_slug` | text | |
| `user_id` | uuid | FK → auth.users |
| `form_hash` | text | SHA-256 of orderRow for idempotency |
| `status` | text | `Unpaid`, `Processing`, `Emailed`, `Shipped`, `Refunded` |
| `stripe_payment_intent` | text | Set by webhook on payment |
| `refunded_cents` | int | Set by webhook on refund |
| `shipping_address` | jsonb | Set by webhook from Stripe session |
| `frame_name` | text | Snapshot from frame at order time |
| `frame_slug` | text | Snapshot |
| `frame_image_src` | text | Snapshot |
| `frame_price_cents` | int | Snapshot |
| `frame_color` | text | Display name, resolved server-side from `frameColorSlug` |
| `total_price_cents` | int | Frame + all addons |
| `deposit_used_cents` | int \| null | Null if no TBYB deposit used |
| `stripe_charge_cents` | int | `max(total - deposit, 50)` |
| `vision_type` | text | |
| `od_sphere/cylinder/axis` | text | |
| `os_sphere/cylinder/axis` | text | |
| `pd_mode` | text | `Single` or `Dual` |
| `pd` | text | Used when `pd_mode = Single` |
| `pd_left` / `pd_right` | text | Used when `pd_mode = Dual` |
| `lens_material` | text | |
| `lens_color_category` | text | Category label (e.g. "Transitions Gen8") |
| `lens_color` | text | Full option name |
| `ar_coating` | text | |
| `scratch_coating` | text | |
| `mirror_coating` | text | |
| `carrier` | text \| null | Set by admin on fulfillment |
| `tracking_number` | text \| null | Set by admin on fulfillment |
| `comments` | text | |
| `prescription_url` | text | Supabase storage URL or `"None"` |
| `headshot_url` | text | Supabase storage URL or `"None"` |
| `contact_name/email/phone` | text | |
| `created_at` | timestamptz | |
| `updated_at` | timestamptz | |

---

## Pricing

All prices are in cents. Total = frame price + sum of addon prices.

Each addon option string is a key in a server-side price map (`VISION_TYPE_PRICES`, `LENS_MATERIAL_PRICES`, etc.). The client sends the exact option string — any mismatch returns a 400. `"None"` and baseline options map to 0.

`stripe_charge_cents = max(total_price_cents - deposit_used_cents, 50)` — minimum 50¢ enforced for Stripe.

---

## Frame Color Resolution

The client sends `frameColorSlug`. The server looks up the frame from `prescription_frames`, finds the matching color entry in the `colors` JSONB array (`{ option: string, slug: string }[]`), and stores `option` (the display name) in `frame_color`. The slug never touches the DB.

---

## TBYB Deposit

If the user provides a `tbybSubmissionId`, the server:

1. Queries the user's own TBYB submissions (RLS-scoped) to find the match by last 8 chars of ID.
2. Computes `available = max(deposit_cents - refunded_cents, 0)`.
3. Validates `submission.depositCents === available && submission.depositCents !== 0` — errors if not equal to available OR if 0 (deposit must be used and must match).
4. Computes `deposit_used_cents = min(available, total_price_cents)`.
5. `deposit_left_cents = deposit_cents - deposit_used_cents` (passed to webhook via metadata for TBYB balance update).

---

## Idempotency

`form_hash` is a SHA-256 of the full `orderRow` object. On insert conflict (`23505`), the server looks up the existing `Unpaid` order with the same hash and reuses its ID. This prevents duplicate orders if the user submits twice before paying.

---

## Stripe Session

- Mode: `payment`
- Charge: `stripe_charge_cents`
- Metadata: `brandSlug`, `type: "rx-order"`, `rxOrderId`, optionally `tbybSubmissionId` and `depositLeftCents`
- Collects: billing address, phone, shipping address (US only)
- `idempotencyKey`: the `orderId` — safe to retry session creation

---

## TBYB Session Race Prevention

If the user has an existing open Stripe session for their TBYB submission:
1. Expire the old session (handle `invalid_request_error` — if it's already complete, return 500 to prevent double-charging).
2. Null out `open_stripe_session_id` on the submission.
3. Re-claim with a conditional update (`.is("open_stripe_session_id", null)`) — if another request claimed it first, return 409.

---

## Webhook Handling

`charge.refunded` → tries `orders` first, then `rx_orders`, then `tbyb_submissions`:

- Sets `refunded_cents = charge.amount_refunded`
- If full refund (`amount_refunded === amount`): sets `status = "Refunded"`

`checkout.session.completed` (type `rx-order`):
- Sets `status = "Processing"`, `stripe_payment_intent`
- Sets `shipping_address` from `session.collected_information.shipping_details`
- If `tbybSubmissionId` in metadata: updates TBYB deposit balance and clears `open_stripe_session_id`

---

## Admin Actions (`src/lib/admin/rx-orders.ts`)

| Function | Description |
|---|---|
| `getRxOrders(brandSlug)` | Fetches all orders, newest first, maps to `RxOrder` type |
| `updateRxStatus(id, status)` | Updates status |
| `saveRxFulfillment(id, carrier, trackingNumber)` | Saves carrier + tracking |
| `undoRxFulfillment(id)` | Nulls carrier + tracking |

---

## Admin UI (`rx-orders-table.tsx`)

### Status Logic

- `STATUSES = ["Processing", "Emailed", "Shipped"]` — dropdown options (Refunded is webhook-only)
- `statusLocked = order.status === "Refunded"` — Unpaid is filtered from visible
- `filterDefs` uses `[...STATUSES, "Refunded"]` + a derived "Partially Refunded" filter (`refundedCents !== null && status !== "Refunded"`)

### Fulfillment Logic

- `shippingLocked = order.status !== "Shipped" || isShipped` — carrier/tracking inputs locked unless status is Shipped and not yet saved
- `saveFulfillmentDisabled` — requires `status === "Shipped"`, both fields filled, no save in flight
- `undoFulfillmentDisabled` — only blocked while a save is in flight; undo is available from any status
- After `handleStatusSave`, draft carrier/trackingNumber reset to saved order values to avoid stale draft display

### Display Details

- PD dual: `L {pdLeft} / R {pdRight}`
- Lens color: `{lensColorCategory} · {lensColor}`
- Deposit: `Deposit {amount}` or `"None"` when null
- Refunded label (expanded panel): brand accent color
- Status dot: black for Shipped/Refunded, brand accent for others
