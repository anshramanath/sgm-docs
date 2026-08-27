# Prescription Frame Orders — Build Reference (August 18, 2026)

## What Was Built

An authenticated, multi-step prescription frame ordering flow at `/rx?tab=frames`. Users select a frame and color, fill out a 6-step form (prescription, PD, lens options, coatings, contact info, review), and are redirected to a Stripe checkout. TBYB customers can optionally apply their deposit to reduce the charge.

---

## Form Flow (`PrescriptionFramesClient.tsx`)

**Step 0** — Optional TBYB order # (exactly 8 chars, last 8 of UUID). Length validated client-side before the API call. If provided, fetches the available deposit via `/api/user/deposit` and stores it in form state and LS. `deposit <= 0` is rejected with an error.

**Step 1** — Prescription. Vision type + OD/OS sphere, cylinder, axis. Axis is disabled when cylinder is `"None"`. Sphere options generated dynamically from the frame's `rxLow`/`rxHigh` range.

**Step 2** — PD. Toggle between Single and Dual (`pdMode` stores `"Single"` or `"Dual"` — user-facing strings). Single sends one `pd`; Dual sends `pdLeft` + `pdRight`. Unused fields sent as `"None"`.

**Step 3** — Lens material and color/type. Radio selects the category (stored as the user-facing label: `"Genuine Transitions"`, `"Polarized"`, `"Solid Tinted"`), which reveals a dropdown for the specific color. Changing category clears the color.

**Step 4** — Coatings. AR coating, scratch coat, mirror coat. All required; `"None"` is a valid option for each.

**Step 5** — Additional info + contact. Optional comments, prescription upload, headshot upload. Required: name and email.

**Step 6** — Review and submit. Calls `/api/user/rx-order`, redirects to Stripe on success.

---

## Frame & Color Selection

Color selection is **ephemeral** — it lives in memory only. The commit point is clicking **"Select Frame."** This is the only action that writes `frameId`/`frameColor` to LS.

Any color swatch click (select or deselect) nulls out `frameId`/`frameColor` in LS unconditionally. This enforces the invariant:

> LS has a frame+color **if and only if** the user clicked "Select Frame."

Refreshing before clicking "Select Frame" intentionally resets to a clean frame grid — no partially-selected state.

Clicking the already-chosen color swatch deselects it (sets `pendingColor` to null) and also clears LS. Clicking any other color updates `pendingColor` in memory but still clears LS since the previously committed frame is now stale.

---

## LocalStorage

Key: `${brandSlug}:rx`

Saved on: step advances, "Select Frame," and any color swatch click.

Shape:
```json
{ "frameId": "uuid-or-null", "frameColor": "slug-or-null", "step": 2, "vals": { ...formVals } }
```

`name` and `email` are excluded from LS (come from the server session on each load).

**On restore (useEffect):**
- `mergedVals` is computed once as `{ ...vals, ...savedVals }` and used for both `setVals` and `saveToLS`.
- Frame found, color slug not found → frame stays null, LS rewritten with null `frameId`/`frameColor`.
- Frame not found → form values and step still restored, LS rewritten with null frame.
- Step preserved as-is — if step is 6 and no sphere conflict on a new frame, the user jumps straight to review.

**On success (`/rx/success`):**
- Only `frameId` and `frameColor` are cleared. `tbybId` and `depositCents` are left intact so the user can pick a new frame and jump to checkout with the deposit still wired up.
- The 422 backend check handles any stale deposit value.

---

## TBYB Deposit System

### DB columns added to `tbyb_submissions`
```sql
alter table tbyb_submissions
  add column deposit_cents int null,
  add column open_stripe_session_id text null;
```

`deposit_cents` is set by the webhook after successful TBYB payment. `open_stripe_session_id` prevents double-spend.

### Flow
1. User enters TBYB order # (8 chars). Frontend validates length, then calls `/api/user/deposit`.
2. `depositCents` stored in form state and LS, sent with the submission.
3. Backend recomputes `available = max(deposit_cents - refunded_cents, 0)`. Returns **422** if `submission.depositCents !== available` or `submission.depositCents === 0`.
4. Frontend handles 422: `freshDeposit <= 0` → clears tbybId and depositCents, shows error. Otherwise updates the stored deposit and shows "changed" message for resubmit.
5. `depositUsed = min(available, totalPriceCents)`. `chargeCents = max(totalPriceCents - depositUsed, 50)`.

### Session locking
- Existing `open_stripe_session_id` is expired before a new session is claimed.
- Optimistic lock: `.update({ open_stripe_session_id: session.id }).is("open_stripe_session_id", null)`. Zero rows → **409**.
- Webhook atomically sets `deposit_cents = depositLeftCents` and `open_stripe_session_id = null`.

---

## Pricing

Backend validates every option against server-side price dicts. Unknown values → 400.

```
totalPriceCents = frame.price_cents
  + visionTypePrice + lensMaterialPrice + lensColorPrice
  + arCoatingPrice + scratchCoatPrice + mirrorCoatPrice
```

`chargeCents = max(totalPriceCents - depositUsed, 50)` — Stripe minimum $0.50.

---

## Key Design Decisions

**User-facing strings as stored values.** No internal codes anywhere. What the user sees is what gets stored in form state, LS, and the DB. Applies to `lensColorCategory`, `pdMode`, and all option fields. No reverse-mapping needed.

**Sentinel `"None"` for optional fields.** Submissions are always strings, never null. Unset optional fields sent as `"None"`: `odAxis`/`osAxis` when cylinder is `"None"`, unused PD fields, coatings, comments, phone, URLs.

**Form state typing.** Text inputs (`name`, `email`, `phone`, `comments`, `tbybId`) are `string`, initialized to `""`. Dropdown fields are `string | null` where null = unselected.

**Idempotency.** Backend hashes the full `orderRow` as SHA-256 `form_hash`. Duplicate insert (unique constraint on `form_hash` where `status = 'Unpaid'`) reuses the existing order ID → same Stripe session via idempotency key.

**Sphere conflict check.** On frame selection, if any existing sphere/cylinder value falls outside the new frame's range, those fields are cleared and step resets to 1. Otherwise step is preserved, allowing a direct jump to review.

**Deposit safety net.** Frontend guards `deposit <= 0` at step 0 and `freshDeposit <= 0` in the 422 handler. Backend guards `submission.depositCents !== available || submission.depositCents === 0`. Multiple layers ensure a zero or stale deposit can never be silently accepted.

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

### `POST /api/user/deposit`
Returns available deposit for a TBYB submission by short ID (last 8 chars, case-insensitive).
- **402** payment not completed · **404** not found
- **200** `{ depositCents: number }`

### `POST /api/user/rx-order`
Submits an Rx order and returns a Stripe checkout URL.
- **400** missing fields or unrecognized lens option
- **404** frame or TBYB submission not found
- **409** session race conflict (retry)
- **422** deposit stale or zero; body includes `{ depositCents: <fresh> }`
- **200** `{ url: string }`

---

## DB Schema

```sql
create table prescription_frames (
  id uuid, brand_slug text, name text, slug text unique,
  image_src text, price_cents int, size text,
  rx_low numeric, rx_high numeric, colors jsonb
);

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

-- Migration on tbyb_submissions
alter table tbyb_submissions
  add column deposit_cents int null,
  add column open_stripe_session_id text null;
```

RLS: users can read their own rx_orders. Admin writes via service role.
