# Orders — Build Notes (July 7, 2026)

Architecture decisions, logic, and patterns from building the admin orders page and its supporting systems.

---

## Overview

The orders page is a client component. `initialOrders` is passed from the server and used exactly once to seed state. All mutable state (carrier, tracking, status, acting) lives in a client-side `Record` keyed by order ID. The server is only touched on Save and Undo.

---

## State Architecture

### The orders Record

```tsx
const [orders, setOrders] = useState<Record<string, Order & { acting: boolean }>>(() =>
  Object.fromEntries(initialOrders.map((o) => [o.id, { ...o, acting: false }]))
);
```

`initialOrders` is only used here. After initialization, everything reads from `orders`. `Object.values(orders)` gives an array for filtering and rendering. The Record gives O(1) updates by ID.

### Page-wide singletons

- `expanded: string | null` — the ID of the currently expanded row. One at a time, so a single value or null is enough.
- `carrierOpen: boolean` — whether the carrier dropdown is open. Since only one row can be expanded, a bool is sufficient — the expanded ID tells you which order it belongs to.

### Per-order acting flag

`acting` lives inside the Record (`orders[id].acting`) instead of a page-level `Set`. Two concurrent saves shouldn't interfere with each other's button state.

### Synthetic filters

Two filter tabs are derived, not DB statuses:

- `all` — ignores status entirely, shows everything
- `partial_refund` — `o.status !== "refunded" && o.refundedCents > 0`

All other filter tabs use `o.status === filter`.

---

## Fulfillment

### Two booleans gate the UI

- `isShipped` (`o.status === "shipped"`) — disables carrier dropdown and tracking input, shows Undo instead of Save
- `isRefunded` (`o.status === "refunded"`) — hides the fulfillment section entirely

`isShipped` is the right check for Save/Undo toggling because `refunded` orders never reach the fulfillment UI at all. Only `processing` and `shipped` render carrier/tracking fields.

### Save flow

```
acting: true → saveFulfillment() → status: "shipped", acting: false
```

Save is disabled unless both `carrier` and `trackingNumber` are non-empty.

### Undo flow

```
acting: true → undoFulfillment() → carrier: null, trackingNumber: null, status: "processing", acting: false
```

### Carrier dropdown backdrop

When `carrierOpen` is true, a `position: fixed; inset: 0; z-index: 9` div is rendered behind the dropdown. Any click outside hits it and closes the dropdown. The dropdown itself sits at `z-index: 10`.

---

## Refund System

### Status model

Three statuses: `processing`, `shipped`, `refunded`

`delivered` and `partially_refunded` were removed. Partial refunds are expressed via `refunded_cents` without changing status.

### Partial vs full refund

| | `refunded_cents` | `status` |
|---|---|---|
| Partial refund | updated | unchanged |
| Full refund | updated | set to `"refunded"` |

A partial refund can happen before or after shipping. The order still ships; carrier and tracking remain valid. If items are returned post-ship and a partial refund is issued, the order stays `shipped` with the refund amount shown in the payment section.

### UI treatment

When `refundedCents > 0`, the payment column in the expanded panel shows:
- Label: "Refunded" (full) or "Partially Refunded" (partial) — brand accent color
- Amount: `formatPrice(o.refundedCents)` — brand accent color

Status badge keeps its own color based on actual status. Accent is not used for status.

---

## DB Trigger

`decrement_total_sales_on_refund` fires on every `UPDATE` to `orders`. Two guards control whether the decrement loop runs:

```sql
-- guard 1: only run if a real positive refund is present
if coalesce(new.refunded_cents, 0) <= 0 then
  return new;
end if;

-- guard 2: only run if we haven't decremented before
if old.refunded_cents > 0 then
  return new;
end if;
```

### Why `coalesce`?

The trigger fires on all updates, including fulfillment saves that don't touch `refunded_cents`. Those leave it as `null`. Without `coalesce`, `null <= 0` evaluates to `null` in SQL (not `true`), so the guard wouldn't fire. `coalesce(null, 0)` → `0` → correctly exits early.

`<= 0` instead of `= 0` also guards against any hypothetical negative value.

### Why `old.refunded_cents > 0`?

`refunded_cents` is either `null` (never refunded) or a positive value set by the webhook — it never passes through `0`. So `old.refunded_cents > 0` means a real refund was already processed and decremented. Using `> 0` rather than `is not null` adds extra safety: a non-positive old value won't be treated as a prior decrement.

### Cases covered

| Scenario | Guard 1 | Guard 2 | Decrement? |
|---|---|---|---|
| Fulfillment update | exits (new = null) | — | No |
| First refund (partial or full) | passes | passes (old = null) | Yes |
| Full refund after partial | passes | exits (old > 0) | No |
| Duplicate Stripe event | passes | exits (old > 0) | No |

The trigger always returns `new` — guards only control the decrement logic, not the row update.

---

## Webhook

```ts
// charge.refunded handler
if (!(charge.amount_refunded > 0)) return new Response("OK", { status: 200 });

const isFullRefund = charge.amount_refunded === charge.amount;

await supabase.from("orders").update({
  refunded_cents: charge.amount_refunded,
  ...(isFullRefund && { status: "refunded" }),
}).eq("stripe_payment_intent", paymentIntent);
```

### The guard

`!(charge.amount_refunded > 0)` rejects `null`, `undefined`, `0`, and negatives. `undefined > 0` is `false` in JS (NaN comparison), so the `!` correctly exits early — unlike `<= 0` which would let `undefined` through.

### Conditional status update

`...(isFullRefund && { status: "refunded" })` only spreads the status field when `amount_refunded === amount`. Partial refunds only update `refunded_cents`.

`charge.amount` is the total charged on the Stripe charge object in cents. Equality with `amount_refunded` means the full charge was returned.

---

## Key Decisions

| Decision | Why |
|---|---|
| `acting` inside Record | Per-order — two concurrent saves shouldn't disable each other's button |
| `expanded` as `string \| null` | Page-wide singleton — one row expands at a time |
| `carrierOpen` as `boolean` | Page-wide singleton — only the expanded row has a dropdown |
| Items count = `o.items.length` | Unique line items, not total quantity |
| Carrier stored as full name | "FedEx", "UPS" — no slug-to-label mapping needed on read |
| `initialOrders` used once | Prop only seeds `useState`; all reads come from `orders` |
| `refunded_cents` never `0` | Webhook guards ensure null → positive; trigger can rely on null/non-null semantics |
