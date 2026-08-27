# Account Page & TBYB History — Build Notes (August 6, 2026)

## What Was Built

TBYB History section on the account page — a list of past Try Before You Buy submissions with an accordion expand/collapse per card showing all form details.

---

## Data

### `TBYBSubmissionRecord` type additions

```ts
refundedCents: number | null;
shippingAddress: ShippingAddress | null;
carrier: string | null;
trackingNumber: string | null;
```

`carrier` and `trackingNumber` are stubbed — UI renders them conditionally but the backend doesn't send them yet.

### Schema additions (`tbyb_submissions`)

```sql
carrier         text,
tracking_number text,
```

(`refunded_cents` and `shipping_address` were added in the prior session.)

### Fetching

```ts
const [orders, submissions] = await Promise.all([getOrders(), getSubmissions()]);
```

Parallel fetch — no reason to wait on one before the other.

---

## `TBYBSubmissions` Component

Client component (`"use client"`) because the accordion toggle needs state.

### Status colors

```ts
const STATUS = {
  Unpaid:   { label: "Unpaid",   color: "#737373" },
  Curating: { label: "Curating", color: "#737373" },
  Emailed:  { label: "Emailed",  color: "#737373" },
  Shipped:  { label: "Shipped",  color: "var(--color-brand)" },
  Received: { label: "Received", color: "#000000" },
  Refunded: { label: "Refunded", color: "#000000" },
};
```

Falls back to grey + raw status string for unknown values.

### Accordion chevron

Up arrow (rotated) when collapsed, down arrow (default) when open:

```tsx
className={`... ${open ? "" : "rotate-180"}`}
```

### Card structure

```
┌─ toggle button (header) ──────────────────────────┐
│  Submission #XXXXXXXX   Date   [Status]  chevron  │
├─ always visible ──────────────────────────────────┤
│  [img]  Package name                              │
│         N Pairs · $X Deposit                      │
│         Refunded $X  (if refundedCents)           │
│                         Carrier · Tracking  (br)  │
├─ expanded panel ──────────────────────────────────┤
│  Left col:  Prescription OD/OS, Lens, Helmet/Hat  │
│  Right col: Nose Bridge, Sunglass Fit, Frame Type │
│             Prescription (link or None)           │
│             Headshot (link or None)               │
│  Additional Info                                  │
│  Contact: name · email · phone                    │
│  Shipping address (or None)                       │
└───────────────────────────────────────────────────┘
```

### "None" display rules

All optional fields are always shown — if the value is `"None"` (string from backend) or `null`, display `None` as text. If a URL, display as a link. This matches the TBYB step 4 review pattern.

- Prescription/Headshot: separate labeled rows, link if URL, "None" if not
- Additional Info: always rendered, shows `"None"` string as-is
- Phone: always rendered inline in contact string
- Shipping address: always shown, `null` → "None"

### Pairs display

```ts
s.packagePairsMin === s.packagePairsMax
  ? s.packagePairsMin
  : `${s.packagePairsMin}–${s.packagePairsMax}`
```

Single number if min === max, range otherwise.

### Carrier / tracking

Conditionally rendered in the bottom-right of the package row:

```tsx
{s.carrier && s.trackingNumber && (
  <p className="text-[13px] shrink-0">
    <span className="text-grey-500">{s.carrier}</span> · {s.trackingNumber}
  </p>
)}
```

`items-end` on the row keeps it bottom-aligned with the package info.

---

## TBYBClient Changes (same commit)

- "Head Shot" → "Headshot" everywhere (label + step 4 row label)
- `saveToLS(toStep: number)` — required param, no default. Every call site is explicit.
  - Step advances: `saveToLS(step + 1)` — saves destination step before React applies state update
  - `selectedPkg` effect: `saveToLS(step)` — saves current step when package is selected/restored
  - Removed from just before Stripe submit — step advances already cover it
- Restore: plain `setStep(savedStep)` — no mapping needed since correct step is always saved
- Result: refresh at any point returns you to exactly where you were

---

## API.md

- `carrier` and `trackingNumber` added to submission record response docs
- Noted: `null` until admin saves shipping info
