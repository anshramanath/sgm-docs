# TBYB Admin — Session Summary (August 6, 2026)

## Files Changed

| File | What changed |
|------|-------------|
| `src/lib/db/004_tbyb.sql` | Added `carrier text`, `tracking_number text` columns |
| `src/lib/db/initial_schema.sql` | Same additions, kept in sync |
| `src/lib/types.ts` | `TbybSubmission` gains `paymentIntent`, `carrier`, `tracking` |
| `src/lib/admin/tbyb.ts` | New server actions, extended select + map |
| `src/app/admin/[brandSlug]/tbyb/tbyb-table.tsx` | Major rewrite of expanded panel + state |
| `src/app/api/user/submissions/route.ts` | Added `carrier`, `trackingNumber` to response |
| `API.md` | Submissions response updated |

---

## DB Migration (run on live DB)

```sql
alter table tbyb_submissions add column carrier text;
alter table tbyb_submissions add column tracking_number text;
```

---

## Server Actions — tbyb.ts

### `updateTbybShipping(id, carrier, tracking)`
Sets `carrier` and `tracking_number` on a submission. Called when the admin saves the fulfillment block.

### `undoTbybShipping(id)`
Sets both to `null`. Named **undo**, not *clear* — matches the intent shown to the user. Reuses `savingShipping` state; no separate undo state needed.

### `updateTbybStatus(id, status)`
Unchanged. Still a separate action — status and shipping save flows are independent.

---

## State Design — tbyb-table.tsx

### `draftFulfillment`
Single `Record<id, { status, carrier, tracking }>` replacing three separate draft states. Updated through a shared `setDraftField` helper:

```ts
setDraftFulfillment((prev) => ({
  ...prev,
  [id]: { ...prev[id], [field]: value }
}))
```

Initialized from `initialSubmissions` — pre-existing carrier/tracking loads into draft via `s.carrier ?? ""`.

### `savingStatus` / `savingShipping`
Two separate loading states, each holding the active submission ID or `null`. They mutually disable each other — you can't save status while shipping is saving, and vice versa.

---

## Fulfillment Logic

### `shippingLocked`
```ts
sub.status !== "Shipped" || !!(sub.carrier && sub.tracking)
```
Inputs lock in two cases: wrong status, or shipping already committed. Once saved, the carrier dropdown and tracking input go read-only — only Undo is interactive.

### `saveShippingDisabled`
```ts
sub.status !== "Shipped"
  || !draft.carrier || !draft.tracking
  || savingShipping !== null || savingStatus !== null
```
Shared by both Save and Undo buttons. Undo is also disabled when status isn't Shipped — by design. No separate `undoDisabled` needed because when the Undo button is visible, draft always has values, so the carrier/tracking conditions don't fire.

### Save / Undo toggle
Keyed off **persisted** state (`sub.carrier && sub.tracking`), not draft. Flips only after a confirmed DB write — not mid-draft.

### Live summary line
Uses `draft.carrier && draft.tracking` for both condition and content — gives the typing effect matching the orders table. Disappears after undo when draft is cleared to `""`.

---

## Layout — Expanded Panel

### Payment / Refunded alignment
Contact and Package columns use `flex-direction: column; justify-content: space-between`. Top content sits at the top; Payment and Refunded always anchor to the bottom, aligned across the row because grid cells share height.

### Payment Intent
Shown in Contact column bottom only when `sub.paymentIntent` is truthy. Field renamed from `stripePaymentIntent` — the prefix was redundant in context.

### Refunded
Shown in Package column bottom only when `sub.refundedCents !== null`. Displayed in brand accent color.

---

## Rules to Remember

### `prescriptionUrl` / `headshotUrl` — use `!== "None"`
These fields store the literal string `"None"` when not provided — never `null`. Always check `!== "None"`, never truthy. Same applies to `contactPhone`.

### `trackingNumber` in API responses, `tracking` in admin types
The user-facing submissions API returns `trackingNumber`. The `TbybSubmission` admin type uses `tracking`. The DB column is always `tracking_number`.

---

## Naming Conventions Settled

| Name | Not |
|------|-----|
| `undoTbybShipping` | ~~clearTbybShipping~~ |
| `savingStatus` / `savingShipping` | ~~saving~~ |
| `shippingLocked` | ~~fulfillmentLocked~~ |
| `saveShippingDisabled` / `saveStatusDisabled` | ~~saveDisabled~~ |
| `paymentIntent` | ~~stripePaymentIntent~~ (in admin type) |
| `draftFulfillment` | umbrella; *shipping* for carrier+tracking ops |
