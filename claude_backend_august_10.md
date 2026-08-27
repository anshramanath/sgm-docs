# Veeqo Order Sync Integration (August 10, 2026)

Automatically creates a Veeqo order whenever a customer completes checkout. Triggered by the Stripe `checkout.session.completed` webhook, runs synchronously before returning 200, and stores the result on the order row.

---

## Table of Contents

1. [Overview](#overview)
2. [Planning Decisions](#planning-decisions)
3. [Stripe Changes](#stripe-changes)
4. [DB Schema](#db-schema)
5. [Veeqo API](#veeqo-api)
6. [CustomerInfo Design](#customerinfo-design)
7. [Idempotency & Race Conditions](#idempotency--race-conditions)
8. [Order Payload](#order-payload)
9. [Error Handling](#error-handling)
10. [Files Changed](#files-changed)
11. [Environment Variables](#environment-variables)
12. [Bugs Fixed During Development](#bugs-fixed-during-development)

---

## Overview

When a customer completes checkout:

1. Stripe fires `checkout.session.completed` to `/api/webhooks/stripe`
2. The webhook inserts the `orders` row and `order_items` rows
3. `syncOrderToVeeqo` is called and **awaited** — the function must complete before returning 200 to Stripe, because Vercel terminates the serverless function the moment the response is sent
4. Veeqo creates the order on their side
5. The result (`veeqo_order_id` on success, `veeqo_error` on failure) is stored on the order row
6. 200 is always returned to Stripe — Veeqo errors are decoupled from the webhook response

---

## Planning Decisions

Every major design choice was made before writing code. This is what made the integration clean.

### Name-Based Resolution Instead of Hardcoded IDs

Veeqo identifies channels and delivery methods by numeric IDs internally, but those IDs are environment-specific and change across accounts. Instead of hardcoding them:

- `VEEQO_CHANNEL_NAME` and `VEEQO_DELIVERY_METHOD_NAME` are stored as env vars with human-readable names
- At sync time, the integration calls `GET /channels` and `GET /delivery_methods`, filters by name, and extracts the ID
- Same philosophy applied to SKUs: `GET /products?query=<sku>` walks `product.sellables` to find the `sellable_id`

This means the integration works across environments and never breaks if IDs change.

### delivery_cost: "0.00"

Shipping is presented to customers as free/included in product pricing. Veeqo's `delivery_cost` is set to `"0.00"` — not omitted, explicitly zero.

### No remote_id on Line Items

Veeqo supports a `remote_id` field on line items to link back to an external system. This was explicitly excluded — pulling a product or variation UUID would require extra DB queries for minimal benefit.

### Always Await the Sync

The sync is awaited before returning the response. Fire-and-forget was considered but rejected: Vercel kills the function on response, so any background work gets terminated. The sync must complete within the request lifecycle.

### Always Return 200 to Stripe

Veeqo and Stripe are decoupled. If Veeqo fails, Stripe should not retry the webhook — that would cause duplicate order inserts. Errors are stored in `veeqo_error` on the order row for manual review. 200 is always returned regardless of Veeqo outcome.

### Atomic Claim to Handle Retries

Stripe can deliver webhooks more than once (retries on timeout, network failure, etc.). Two simultaneous invocations for the same order could both attempt to create a Veeqo order. Solved with a conditional UPDATE — see [Idempotency](#idempotency--race-conditions).

---

## Stripe Changes

### Phone Number Collection

```ts
phone_number_collection: { enabled: true }
```

Added to both `/api/user/checkout` and `/api/user/tbyb`. Makes `session.customer_details.phone` available in the webhook, which is required for Veeqo's `deliver_to_attributes.phone`.

### Billing Address Collection

```ts
billing_address_collection: "required"
```

Without this, Stripe defaults to `"auto"` — collecting billing only when the payment method requires it, which means only country and zip are collected. With `"required"`, Stripe always collects the full billing address. This is what populates `session.customer_details.address` with all fields (line1, city, state, postal_code, country), which is then passed to Veeqo's `billing_address_attributes`.

Note: when a customer selects "same as shipping", Stripe copies all shipping fields into the billing address — full address either way.

### Removed automatic_tax

`automatic_tax: { enabled: true }` was briefly added but caused 500 errors from Stripe because Stripe Tax had not been configured in the dashboard. Removed from both checkout routes.

---

## DB Schema

Two columns added to the `orders` table:

```sql
veeqo_order_id  bigint  null,
veeqo_error     text    null,
```

- `veeqo_order_id` — the Veeqo order ID on success
- `veeqo_error` — error message on failure, or `"Sync in progress"` while the sync is running (acts as a distributed lock — see below)

Both start as `null`. After a successful sync, `veeqo_order_id` is set and `veeqo_error` is cleared to `null`. After a failed sync, `veeqo_order_id` stays `null` and `veeqo_error` holds the reason.

Migration: `src/lib/db/003_orders.sql`  
Also reflected in: `src/lib/db/initial_schema.sql`

---

## Veeqo API

Base URL: `https://api.veeqo.com`  
Auth: `x-api-key: <VEEQO_SECRET_KEY>` header on every request

A shared `veeqo()` helper wraps `fetch` and injects the base URL and headers:

```ts
function veeqo(path: string, options?: RequestInit) {
  return fetch(`https://api.veeqo.com${path}`, {
    ...options,
    headers: {
      "x-api-key": process.env.VEEQO_SECRET_KEY!,
      "Content-Type": "application/json",
    },
  });
}
```

### Resolving Channel ID

```ts
GET /channels
```

Filter: `c.name === VEEQO_CHANNEL_NAME && c.type_code === "direct" && c.state === "active"`

Throws if zero or multiple matches — prevents silent misconfiguration.

### Resolving Delivery Method ID

```ts
GET /delivery_methods
```

Filter: `m.name === VEEQO_DELIVERY_METHOD_NAME`

### Resolving SKU → Sellable ID

```ts
GET /products?query=<encodeURIComponent(sku)>
```

`encodeURIComponent` converts spaces and special characters to URL-safe encoding (e.g. `%20`). The server decodes them back — it's a transport requirement, not a data transformation.

Veeqo's response uses `product.sellables` (not `variants`). Each sellable has a `sku_code` and an `id`. Walk all products and all sellables, match on `sku_code`, collect the `id`.

```ts
for (const product of products) {
  for (const sellable of product.sellables ?? []) {
    if (sellable.sku_code === sku) matches.push(sellable.id);
  }
}
```

Throws if zero or multiple matches.

### Concurrency

Channel, delivery method, and all SKU resolutions that can run in parallel do so via `Promise.all`:

```ts
const [channelId, deliveryMethodId] = await Promise.all([
  resolveChannel(),
  resolveDeliveryMethod(),
]);

const lineItems = await Promise.all(
  items.map(async (item) => ({
    sellable_id: await resolveSku(item.sku),
    ...
  }))
);
```

---

## CustomerInfo Design

The type evolved through several iterations to cleanly separate billing (who's paying) from shipping (where to send).

```ts
type Address = {
  line1: string | null | undefined;
  line2: string | null | undefined;
  city: string | null | undefined;
  state: string | null | undefined;
  postal_code: string | null | undefined;
  country: string | null | undefined;
};

type CustomerInfo = {
  email: string | null;
  phone: string | null;
  billing: {
    name: string | null;  // session.customer_details.name
    address: Address;     // session.customer_details.address (guaranteed with billing_address_collection: "required")
  };
  name: string | null;    // session.collected_information.shipping_details.name
  address: Address;       // session.collected_information.shipping_details.address
};
```

### Data Sources in the Webhook

| Field | Stripe source |
|-------|---------------|
| `email` | `session.customer_email` (always set — passed at session creation) |
| `phone` | `session.customer_details.phone` |
| `billing.name` | `session.customer_details.name` |
| `billing.address` | `session.customer_details.address` |
| `name` | `session.collected_information.shipping_details.name` |
| `address` | `session.collected_information.shipping_details.address` |

### Veeqo Mapping

| CustomerInfo field | Veeqo field |
|--------------------|-------------|
| `billing.*` | `customer_attributes.billing_address_attributes` |
| `phone` | `customer_attributes.phone` |
| `email` | `customer_attributes.email` |
| `name` | `deliver_to_attributes.first_name` |
| `address.*` | `deliver_to_attributes.address1/city/state/zip/country` |
| `email` | `deliver_to_attributes.email` |
| `phone` | `deliver_to_attributes.phone` |

Note: Veeqo uses `first_name` not `name`, and `zip` not `postal_code`. Stripe's full name from checkout is passed as `first_name` with no `last_name`.

---

## Idempotency & Race Conditions

### The Problem

Stripe retries webhooks on timeout or network failure. Two invocations for the same `checkout.session.completed` event could both pass the order upsert (which is idempotent on `stripe_session_id`) and then both attempt to create a Veeqo order — producing a duplicate.

### Solution: Atomic Conditional UPDATE

```sql
UPDATE orders
SET    veeqo_error = 'Sync in progress'
WHERE  id = $orderId
  AND  veeqo_order_id IS NULL
  AND  veeqo_error IS NULL
RETURNING id
```

Postgres evaluates the `WHERE` clause and the `UPDATE` atomically within a row lock. Only one concurrent invocation can win — the other sees 0 rows updated (`.maybeSingle()` returns `null`) and returns early.

```ts
const { data: claimed, error: claimError } = await supabase
  .from("orders")
  .update({ veeqo_error: "Sync in progress" })
  .eq("id", orderId)
  .is("veeqo_order_id", null)
  .is("veeqo_error", null)
  .select("id")
  .maybeSingle();

if (claimError) console.error("Veeqo sync claim DB error", claimError);
if (claimError || !claimed) return;
```

**`claimError`** — a DB-level failure (connection error, query error). Logged.  
**`!claimed`** — 0 rows updated: either another invocation already claimed it, or the sync already completed/failed. Silent return.  
**`.maybeSingle()`** — returns `null` for 0 rows (valid), a row for 1 row. `.single()` would throw on 0 rows, incorrectly setting `claimError = true` for a legitimate "already handled" case.

### Sentinel as Lock

`veeqo_error = "Sync in progress"` serves as an in-flight lock. If the process crashes mid-sync, this value stays on the row — the order is marked as errored and won't be retried automatically (preventing duplicate Veeqo orders). Manual intervention is required to clear it and retry.

---

## Order Payload

```ts
{
  order: {
    channel_id: channelId,
    delivery_method_id: deliveryMethodId,
    number: `${brandName} - #${orderId.slice(-8).toUpperCase()}`,
    delivery_cost: "0.00",
    customer_attributes: {
      email: customer.email,
      phone: customer.phone,
      billing_address_attributes: {
        first_name: customer.billing.name,
        address1: customer.billing.address.line1,
        address2?: customer.billing.address.line2,  // omitted if null
        city: customer.billing.address.city,
        state: customer.billing.address.state,
        zip: customer.billing.address.postal_code,
        country: customer.billing.address.country,
      },
    },
    deliver_to_attributes: {
      first_name: customer.name,
      address1: customer.address.line1,
      address2?: customer.address.line2,            // omitted if null
      city: customer.address.city,
      state: customer.address.state,
      zip: customer.address.postal_code,
      country: customer.address.country,
      email: customer.email,
      phone: customer.phone,
    },
    line_items_attributes: [
      {
        sellable_id: <resolved from SKU>,
        quantity: item.qty,
        price_per_unit: (item.priceCents / 100).toFixed(2),
      }
    ],
    payment_attributes: {
      payment_type: "credit_card",
      reference_number: paymentIntent,  // Stripe PaymentIntent ID
    },
  }
}
```

### Order Number Format

```
{BrandName} - #{orderId[-8:].toUpperCase()}
```

e.g. `BikerShades - #AEC1E1B6`

Brand name is resolved via `getBrandBySlug(brandSlug)?.name ?? brandSlug` from `src/lib/brand.ts`. Falls back to the slug if the brand isn't found.

---

## Error Handling

```ts
try {
  // ... all Veeqo API calls ...
  veeqoOrderId = data.id;
} catch (e) {
  veeqoError = typeof e === "string" ? e : "Veeqo order creation failed";
}

// Always runs — writes success or failure to the DB
await supabase
  .from("orders")
  .update({ veeqo_order_id: veeqoOrderId, veeqo_error: veeqoError })
  .eq("id", orderId);
```

All thrown errors inside the try block are strings (e.g. `"SKU not found in Veeqo: ABC123"`) so the catch preserves the message directly. Non-string throws (unexpected JS errors) fall back to the generic message.

The final update always runs regardless of outcome, clearing `"Sync in progress"` and writing either the Veeqo order ID or the error reason.

---

## Files Changed

| File | What Changed |
|------|-------------|
| `src/lib/veeqo.ts` | New file — `veeqo()` helper, `resolveChannel`, `resolveDeliveryMethod`, `resolveSku`, `CustomerInfo` type, `syncOrderToVeeqo` |
| `src/app/api/webhooks/stripe/route.ts` | Import `syncOrderToVeeqo`, pass billing + shipping from session, call after items upsert |
| `src/app/api/user/checkout/route.ts` | Added `phone_number_collection`, `billing_address_collection`, removed `automatic_tax` |
| `src/app/api/user/tbyb/route.ts` | Same as checkout |
| `src/lib/db/003_orders.sql` | `ALTER TABLE orders ADD COLUMN veeqo_order_id bigint null, veeqo_error text null` |
| `src/lib/db/initial_schema.sql` | Same columns in base schema definition |
| `API.md` | Webhook order case, billing/shipping split for Veeqo, updated checkout/tbyb descriptions |

---

## Environment Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `VEEQO_SECRET_KEY` | Veeqo API key | — |
| `VEEQO_CHANNEL_NAME` | Exact name of the Veeqo sales channel | `BikerShades-proSport-SGM` |
| `VEEQO_DELIVERY_METHOD_NAME` | Exact name of the Veeqo delivery method | `USPS Ground Advantage` |

Set in `.env.local` (local) and Vercel dashboard (production). The SQL migration for `veeqo_order_id` / `veeqo_error` was also run in the Supabase SQL editor.

---

## Bugs Fixed During Development

These came up during implementation and are worth knowing for future Veeqo work.

**`product.variants` vs `product.sellables`**  
Initial code used `product.variants` with `variant.sellable_id ?? variant.id`. The actual Veeqo API response uses `product.sellables` with `sellable.id` directly. Confirmed via test data showing the correct field structure.

**`name` vs `first_name` in deliver_to_attributes**  
Veeqo requires `first_name`, not `name`. Using `name` silently failed.

**Missing `payment_type` in payment_attributes**  
Initial payload only sent `reference_number`. Veeqo also requires `payment_type: "credit_card"`. Discovered from a failed test order.

**`billing_address_collection` and partial address**  
Without `"required"`, Stripe only provides country + zip in `customer_details.address`. All other fields (`line1`, `city`, `state`) are null. Adding `"required"` ensures the full address is always collected — both when the customer enters it manually and when they select "same as shipping."

**`automatic_tax` causing 500 errors**  
Adding `automatic_tax: { enabled: true }` to Stripe checkout sessions caused 500 responses because Stripe Tax was not configured in the Stripe dashboard. Removed from both checkout routes.

**`.single()` vs `.maybeSingle()` on the claim query**  
`.single()` throws when 0 rows are returned — which happens legitimately when the race is lost or the sync already ran. Using `.single()` would set `claimError = true` for a valid "already handled" case. `.maybeSingle()` returns `null` for 0 rows without error.
