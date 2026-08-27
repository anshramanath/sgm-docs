# Prescription Frame Orders — Build Reference (August 16, 2026)

## What Was Built

An authenticated, multi-step prescription frame ordering flow at `/rx?tab=frames`. Users select a frame and color, fill out a 6-step form (prescription, PD, lens options, coatings, contact info, review), and are redirected to a Stripe checkout. TBYB (Try Before You Buy) customers can optionally apply their deposit to reduce the charge.

---

## Form Flow (`PrescriptionFramesClient.tsx`)

**Step 0** — Optional TBYB order # (8 chars, last 8 of UUID). If provided, fetches the available deposit via `/api/user/deposit` and stores it in form state and localStorage.

**Step 1** — Prescription. Vision type + OD/OS sphere, cylinder, axis. Axis is disabled when cylinder is `"None"`. Sphere options are dynamically generated from the frame's `rxLow`/`rxHigh` range.

**Step 2** — PD. Toggle between Single and Dual. Single sends one `pd` value; Dual sends `pdLeft` + `pdRight`. Unused fields are sent as `"None"`.

**Step 3** — Lens material and color/type. Radio selects the category (Genuine Transitions / Polarized / Solid Tinted), which reveals a dropdown for the specific color. Changing category clears the color selection.

**Step 4** — Coatings. AR coating, scratch coat, mirror coat. All required; `"None"` is a valid option for each.

**Step 5** — Additional info + contact. Optional comments, prescription upload, headshot upload. Required: name and email.

**Step 6** — Review and submit. Calls `/api/user/rx-order`, redirects to Stripe on success.

---

## LocalStorage

Key: `${brandSlug}:rx`

Saved on every step advance and frame selection. Shape:
```json
{ "frameId": "uuid", "frameColor": "slug", "step": 2, "vals": { ...formVals } }
```

`name` and `email` are excluded from LS (they come from the server on each load). On restore:
- `mergedVals` is computed once as `{ ...vals, ...savedVals }` and used for both `setVals` and `saveToLS`.
- If the frame is found but the color slug is not, the frame is set to null and LS is rewritten with `null` frameId — the user must re-select the frame.
- If the frame is not found, form values and step are still restored.

---

## TBYB Deposit System

### DB columns added to `tbyb_submissions`
```sql
alter table tbyb_submissions
  add column deposit_cents int null,
  add column open_stripe_session_id text null;
```

`deposit_cents` is set by the webhook after successful TBYB payment. `open_stripe_session_id` tracks which Stripe session is currently open for a deposit spend, preventing double-spend.

### Flow
1. User enters their TBYB order # (8 chars). Frontend validates length, then calls `/api/user/deposit` to get `depositCents`.
2. `depositCents` is stored in form state and sent with the submission.
3. Backend recomputes `available = max(deposit_cents - refunded_cents, 0)`. If `submission.depositCents !== available`, returns **422** with `{ depositCents: available }`.
4. Frontend handles 422: if `freshDeposit <= 0`, clears the TBYB ID and shows an error; otherwise updates the stored deposit and shows a "changed" message for the user to review and resubmit.
5. `depositUsed = min(available, totalPriceCents)`. `chargeCents = max(totalPriceCents - depositUsed, 50)` (Stripe minimum).

### Session locking
- If the TBYB submission already has an `open_stripe_session_id`, the old session is expired before a new one is claimed.
- New session is claimed with an optimistic lock: `.update({ open_stripe_session_id: session.id }).eq("id", ...).is("open_stripe_session_id", null)`. Zero rows updated → **409** conflict.
- Webhook (on `rx-order` completion) atomically sets `deposit_cents = depositLeftCents` and `open_stripe_session_id = null`.

---

## Pricing

Backend validates every option against server-side price dicts. Unknown values return 400.

```
totalPriceCents = frame.price_cents + visionTypePrice + lensMaterialPrice + lensColorPrice
                + arCoatingPrice + scratchCoatPrice + mirrorCoatPrice
```

Addons with no extra cost (e.g. Polycarbonate, Clear / No Tint, None coatings) are `0` in the dicts.

---

## Key Design Decisions

**User-facing strings as stored values.** `lensColorCategory`, `pdMode`, and all option dropdowns store the exact user-facing label. No internal codes. This means:
- No mapping needed between internal and display values.
- The DB is human-readable.
- What the frontend sends is what the backend stores and validates against.

`lensColorCategory` values: `"Genuine Transitions"`, `"Polarized"`, `"Solid Tinted"`
`pdMode` values: `"Single"`, `"Dual"`

**Sentinel `"None"` for optional fields.** Submission fields are always strings — never `null`. Optional fields that weren't filled in are sent as `"None"`. This applies to: `odAxis`/`osAxis` when cylinder is `"None"`, unused PD fields, `comments`, `phone`, `prescriptionUrl`, `headshotUrl`, coating fields when none is chosen.

**Form state typing.** Text inputs (`name`, `email`, `phone`, `comments`, `tbybId`) are typed `string` and initialized to `""` — React controlled inputs require a string value. Dropdown fields are `string | null` where `null` means unselected (the `Dropdown` component renders "Select" for null).

**Idempotency.** The backend hashes the full `orderRow` as a SHA-256 `form_hash`. On duplicate insert (unique constraint on `form_hash` where `status = 'Unpaid'`), it looks up and reuses the existing order ID and returns the same Stripe session (via idempotency key = `orderId`).

---

## Types (`src/lib/types.ts`)

```ts
type RxFrameOrderResult =
  | { url: string }
  | { data: { depositCents: number }; status: number };

type RxFrameSubmission = {
  frameId: string; frameColorSlug: string;
  tbybSubmissionId: string | null; depositCents: number | null;
  visionType: string;
  odSphere: string; odCylinder: string; odAxis: string;
  osSphere: string; osCylinder: string; osAxis: string;
  pdMode: string; pd: string; pdLeft: string; pdRight: string;
  lensMaterial: string; lensColorCategory: string; lensColor: string;
  arCoating: string; scratchCoating: string; mirrorCoating: string;
  comments: string; prescriptionUrl: string; headshotUrl: string;
  name: string; email: string; phone: string;
};

type TBYBDepositInfo = { depositCents: number };
```

---

## API

### `GET /api/user/deposit`
Returns available deposit for a TBYB submission by short ID (last 8 chars of UUID, case-insensitive).
- **402** — TBYB payment not completed
- **404** — submission not found
- **200** `{ depositCents: number }`

### `POST /api/user/rx-order`
Submits an Rx order and returns a Stripe checkout URL.
- **400** — missing fields or unrecognized lens option
- **404** — frame or TBYB submission not found
- **409** — session race conflict (retry)
- **422** — deposit amount changed; body includes `{ depositCents: <fresh> }`
- **200** `{ url: string }`

---

## DB Schema

```sql
-- Catalog
create table prescription_frames (
  id uuid, brand_slug text, name text, slug text unique,
  image_src text, price_cents int, size text,
  rx_low numeric, rx_high numeric, colors jsonb
);

-- Orders
create table rx_orders (
  id uuid, brand_slug text, user_id uuid,
  frame_name text, frame_slug text, frame_image_src text,
  frame_price_cents int, frame_color text,
  total_price_cents int, deposit_used_cents int, stripe_charge_cents int,
  status text, stripe_session_id text, stripe_payment_intent text,
  form_hash text not null,  -- unique index where status = 'Unpaid'
  vision_type text, od_sphere text, od_cylinder text, od_axis text,
  os_sphere text, os_cylinder text, os_axis text,
  pd_mode text, pd text, pd_left text, pd_right text,
  lens_material text, lens_color_category text, lens_color text,
  ar_coating text, scratch_coating text, mirror_coating text,
  carrier text, tracking_number text,
  comments text, prescription_url text, headshot_url text,
  contact_name text, contact_email text, contact_phone text
);
```

RLS: users can read their own orders. Admin writes via service role.
