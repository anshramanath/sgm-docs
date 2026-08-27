# Rx Order History — Account Page (August 19, 2026)

Reference for the Rx order history display on the customer account page.

---

## Files

| File | Role |
|------|------|
| `src/app/(bare)/account/page.tsx` | Server component — fetches rx orders, renders section |
| `src/app/(bare)/account/RxOrders.tsx` | Client component — expandable card UI |
| `src/lib/types.ts` | `RxOrder` type |
| `src/lib/api.ts` | `getRxOrders()` |

---

## API

**Endpoint:** `POST /api/user/rx-orders`  
**Body:** `{ brandSlug }`  
**Auth:** Bearer token required  
**Returns:** `RxOrder[]` ordered by `created_at DESC`

```ts
export async function getRxOrders(): Promise<RxOrder[]>
```

401 → redirect `/sign-in`. 500/503 → throw. Default → redirect `/try-again`.

---

## RxOrder Type

```ts
export type RxOrder = {
  id: string;
  status: string;
  frameName: string; frameImageSrc: string; frameColor: string;
  framePriceCents: number;
  totalPriceCents: number; depositUsedCents: number | null; stripeChargeCents: number; refundedCents: number | null;
  carrier: string | null; trackingNumber: string | null;
  visionType: string;
  odSphere: string; odCylinder: string; odAxis: string;
  osSphere: string; osCylinder: string; osAxis: string;
  pdMode: string; pd: string; pdLeft: string; pdRight: string;
  lensMaterial: string; lensColorCategory: string; lensColor: string;
  arCoating: string; scratchCoating: string; mirrorCoating: string;
  comments: string; prescriptionUrl: string; headshotUrl: string;
  contactName: string; contactEmail: string; contactPhone: string;
  shippingAddress: ShippingAddress | null;
  createdAt: string;
};
```

**Key distinctions:**
- `framePriceCents` — frame cost alone, shown in the summary row next to color
- `totalPriceCents` — frame + lens addons, shown in the Payment row
- `depositUsedCents` — null when no TBYB deposit was applied
- `refundedCents` — null when no refund, shown regardless of status (partial refunds can occur on Shipped orders)

---

## Status Map

| Status | Color | Meaning |
|--------|-------|---------|
| Unpaid | grey | Payment not completed — filtered out on account page |
| Processing | grey | Payment confirmed, being processed |
| Emailed | grey | Customer has been contacted about their order |
| Shipped | brand | Lenses shipped |
| Refunded | black | Order refunded |

Unknown statuses fall through to `?? { label: o.status, color: "#737373" }`.

---

## Component Structure (`RxOrders.tsx`)

### Header row (always visible, clickable)
```
Rx Order #XXXXXXXX    Created [date]    [STATUS BADGE]    [chevron]
```

### Summary row (always visible)
```
[frame image]  Frame Name
               Frame Color · $framePriceCents
               Refunded $x.xx              ← only if refundedCents !== null
                                           [Carrier · tracking]  ← only if both present
```

### Expanded panel (on open)

**Left column:**
- Prescription — OD (Right): `Sphere · Cylinder · Axis`
- Prescription — OS (Left): `Sphere · Cylinder · Axis`
- PD: single value or `Left x / Right y` when `pdMode === "Dual"`
- Vision Type
- Lens Material

**Right column:**
- Lens Color: `{lensColorCategory} — {lensColor}`
- AR Coating
- Scratch Coating
- Mirror Coating
- Prescription: link or "None"
- Headshot: link or "None"

**Full-width rows below:**
- Additional Info
- Payment: `Total $x · Deposit $x · Charged $x` — deposit defaults to `$0.00` when null
- Contact: `name · email · phone` then address on next line (or "None")

---

## Key Design Decisions

**`framePriceCents` vs `totalPriceCents` in summary**  
Summary row shows `framePriceCents` (frame cost alone) — closer to what the user selected. `totalPriceCents` (includes lens addons) appears in the Payment row where the full breakdown is shown.

**Refunded display is status-agnostic**  
`refundedCents !== null` guard is used, not `status === "Refunded"`. A partial refund can happen on a Shipped order.

**Deposit always shown in payment**  
`depositUsedCents ?? 0` — always renders `Total · Deposit · Charged` as one line. Orders without a TBYB deposit show `Deposit $0.00`.

**Unpaid filtered at call site**  
`rxOrders.filter(o => o.status !== "Unpaid")` in `page.tsx`, same pattern as TBYB submissions.

**`eyeRow` matches TBYB exactly**  
No special handling for "None" axis — displayed as-is.

**Contact phone displayed unconditionally**  
Matches TBYB pattern. Phone value is `"None"` when not provided.

**Address format**  
`line1[, line2], city, state postalCode` — shows `"None"` when `shippingAddress` is null. Matches TBYB exactly.

---

## Alignment with TBYBSubmissions

Both components share the same:
- Card border/padding structure
- Header row layout (id + date + status badge + chevron)
- Summary row layout (image + name + detail + tracking)
- `eyeRow` function signature and output
- `dt`/`dd` label/value pattern in expanded panel
- `prescriptionUrl`/`headshotUrl` link-or-None pattern
- Additional Info, Contact, Address sections

Differences:
- RxOrders has Payment row (TBYB doesn't)
- RxOrders has more expanded fields (lens, coatings, PD, vision type)
- Summary subtitle is `frameColor · framePriceCents` vs TBYB's `pairs · deposit`
