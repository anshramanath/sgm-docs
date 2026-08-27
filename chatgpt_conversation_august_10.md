# Stripe → Supabase → Veeqo Order Sync (August 10, 2026)

## Complete Build, Decisions, Tests, and Lessons Learned

This document captures everything learned and built while integrating
paid Stripe Checkout orders with Supabase and Veeqo for the shared
sunglasses e-commerce backend.

------------------------------------------------------------------------

## 1. Final Goal and Flow

The goal was to automatically create a Veeqo order after a successful
normal Stripe Checkout purchase.

``` text
Customer completes Stripe Checkout
        ↓
Stripe sends checkout.session.completed
        ↓
Next.js webhook verifies Stripe signature
        ↓
Order upserted into Supabase
        ↓
Order items upserted into Supabase
        ↓
await syncOrderToVeeqo(...)
        ↓
Atomically claim Veeqo sync
        ↓
Resolve channel + delivery method + SKU sellable IDs
        ↓
POST /orders to Veeqo
        ↓
Success → save veeqo_order_id
Failure → save veeqo_error
        ↓
Return 200 to Stripe
```

Supabase remains the application's own order database. Veeqo is the
inventory and fulfillment representation.

------------------------------------------------------------------------

## 2. Files Changed

### Database

`src/lib/db/003_orders.sql`

`src/lib/db/initial_schema.sql`

Added:

``` sql
veeqo_order_id bigint null,
veeqo_error text null
```

### Environment

`.env.local`

``` env
VEEQO_SECRET_KEY=<actual Veeqo API key>
VEEQO_CHANNEL_NAME=BikerShades-proSport-SGM
VEEQO_DELIVERY_METHOD_NAME=USPS Ground Advantage
```

Quotes are not required around `USPS Ground Advantage`.

### Veeqo integration

`src/lib/veeqo.ts`

Contains: - Veeqo request helper - channel resolution - delivery-method
resolution - SKU → sellable resolution - atomic sync claim - Veeqo order
creation - final success/error persistence

### Stripe webhook

`src/app/api/webhooks/stripe/route.ts`

Contains: - Stripe webhook signature verification - TBYB handling -
normal order handling - local order/item upserts - awaited Veeqo sync -
refund handling

------------------------------------------------------------------------

## 3. Why Sync from the Stripe Webhook

`checkout.session.completed` is the trusted backend confirmation that
Checkout completed.

The frontend should not decide whether a Veeqo order exists.

The order path is therefore:

``` text
Stripe webhook
→ local order
→ local items
→ Veeqo
```

The Veeqo sync is awaited:

``` ts
await syncOrderToVeeqo(...);
return new Response("OK", { status: 200 });
```

### Why not fire and forget?

The route and `syncOrderToVeeqo()` execute inside the same serverless
invocation. Moving code to another `.ts` file does not give it an
independent process.

Once the HTTP response is returned, un-awaited background work should
not be relied upon to continue.

This is different from a browser firing a request to a backend. Once the
backend receives that request, the backend owns its execution even if
the browser later closes.

------------------------------------------------------------------------

## 4. Serverless, Invocations, Instances, and Concurrency

Serverless means we do not maintain one permanently running application
server ourselves. The platform runs compute as needed.

An invocation is one execution of the serverless function for a request.

``` text
Webhook #1 → Invocation #1
Webhook #2 → Invocation #2
Webhook #3 → Invocation #3
```

Different invocations may run concurrently and may run on different
compute instances.

One Node instance can also have many requests in progress concurrently
because most backend work is I/O:

``` text
Request A → Supabase → waits
Request B → Stripe → waits
Request C → Veeqo → waits
```

While one request waits on the network, the event loop can progress
another.

### Concurrency vs. parallelism

Concurrency means multiple tasks are in progress. One thread can provide
concurrency.

Parallelism means multiple tasks are literally executing simultaneously
on separate hardware execution resources.

A thread is software execution state. A CPU core is hardware. They are
not the same thing.

This matters because two webhook invocations can reach the same order
concurrently.

------------------------------------------------------------------------

## 5. Local Stripe Idempotency

Orders are upserted using:

``` ts
.upsert(orderData, { onConflict: "stripe_session_id" })
```

so Stripe retrying the same Checkout Session does not intentionally
create another local order.

Order items use:

``` ts
.upsert(orderItems, { onConflict: "order_id,sku" })
```

with a unique constraint on `(order_id, sku)`.

That makes local order-item writes retry-safe.

However, this alone does not prevent duplicate Veeqo POSTs.

------------------------------------------------------------------------

## 6. The Veeqo Idempotency Race

A naive implementation could do:

``` text
Invocation A                    Invocation B

read veeqo_order_id = null      read veeqo_order_id = null
read veeqo_error = null         read veeqo_error = null

decide to sync                  decide to sync

POST Veeqo                      POST Veeqo
```

Both invocations saw the same unhandled state before either wrote
anything.

A simple earlier check like:

``` ts
if (veeqo_order_id || veeqo_error) return;
```

does not solve this concurrent check-then-act race.

------------------------------------------------------------------------

## 7. Atomic Claim

The final implementation claims the sync using one conditional UPDATE:

``` ts
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

The check and mutation are one database operation.

``` text
A ─┐
   ├→ UPDATE same row WHERE id/error are null
B ─┘
             ↓
         PostgreSQL
             ↓
only one invocation changes
NULL → "Sync in progress"
```

The winner receives the row and continues.

The loser updates zero rows and exits.

This does not lock the entire database. PostgreSQL coordinates the
relevant row/write.

------------------------------------------------------------------------

## 8. Sync State Model

The two columns now represent:

``` text
veeqo_order_id = null
veeqo_error = null
→ not handled yet

veeqo_order_id = null
veeqo_error = "Sync in progress"
→ claimed
→ if it remains here, execution probably stopped unexpectedly

veeqo_order_id = null
veeqo_error = "<message>"
→ failed and requires investigation

veeqo_order_id = <Veeqo ID>
veeqo_error = null
→ success
```

`"Sync in progress"` acts as both the concurrency claim and a useful
crash marker if execution never reaches the final update.

------------------------------------------------------------------------

## 9. Hardest Distributed Idempotency Case

The dangerous edge case is:

``` text
POST /orders sent
        ↓
Veeqo creates the order
        ↓
backend dies
        ↓
Supabase never saves Veeqo ID
```

Local state remains:

``` text
veeqo_order_id = null
veeqo_error = "Sync in progress"
```

Automatically retrying could create another Veeqo order.

The current design intentionally does not retry this uncertain state
automatically. It flags it for manual investigation.

This is conservative but protects against duplicate remote orders.

------------------------------------------------------------------------

## 10. Veeqo API Helper

``` ts
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

The secret key remains backend-only.

------------------------------------------------------------------------

## 11. Dynamic Channel Resolution

The app calls:

``` text
GET /channels
```

and finds exactly one channel satisfying:

``` ts
c.name === process.env.VEEQO_CHANNEL_NAME
c.type_code === "direct"
c.state === "active"
```

Configuration:

``` text
BikerShades-proSport-SGM
```

Behavior:

``` text
0 matches → Sales channel not found or inactive
1 match   → use its ID
>1        → Multiple active sales channels matched
```

We deliberately resolve by the resource identity rather than permanently
hardcoding a numeric ID. If the ID changes but the intended active
channel still exists under the configured identity, the integration can
continue working.

------------------------------------------------------------------------

## 12. Dynamic Delivery Method Resolution

The app calls:

``` text
GET /delivery_methods
```

and finds exactly one method whose name equals:

``` text
USPS Ground Advantage
```

Behavior:

``` text
0 matches → Delivery method not found
1 match   → use its ID
>1        → Multiple delivery methods matched
```

Again, the meaningful configuration is the desired method, not a fragile
numeric ID.

------------------------------------------------------------------------

## 13. SKU Resolution

For each local SKU:

``` text
GET /products?query=<encoded SKU>
```

The important discovery was that Veeqo's actual SKU/sellable
relationship is under:

``` ts
product.sellables
```

not `product.variants`.

Correct implementation:

``` ts
const matches: number[] = [];

for (const product of products) {
  for (const sellable of product.sellables ?? []) {
    if (sellable.sku_code === sku) {
      matches.push(sellable.id);
    }
  }
}

if (matches.length === 0) {
  throw `SKU not found in Veeqo: ${sku}`;
}

if (matches.length > 1) {
  throw `Multiple Veeqo products matched SKU: ${sku}`;
}

return matches[0];
```

### Why exact matching?

`?query=` is a search. It can return related products.

The integration therefore searches the returned objects for:

``` ts
sellable.sku_code === sku
```

before accepting anything.

------------------------------------------------------------------------

## 14. Sellables vs. Channel Duplicates

We investigated apparent duplicates in Veeqo.

The same inventory sellable can have multiple channel mappings:

``` text
Underlying sellable
ID 426000363
SKU FO60X045BD-BK-SMPOL-MFLG-ST-3
        │
        ├── channel mapping
        ├── another channel mapping
        └── another channel mapping
```

Those are not separate inventory sellables.

A direct API test confirmed that the tested exact SKU resolved to one
underlying sellable ID:

``` text
426000363
```

The implementation iterates `product.sellables`, not
`channel_sellables`, so the channel mappings do not create false SKU
ambiguity.

If two genuinely distinct `sellable.id` values ever have the exact same
SKU, the integration intentionally errors rather than arbitrarily
picking the first inventory record.

------------------------------------------------------------------------

## 15. Line Item Mapping

Each local item becomes:

``` ts
{
  sellable_id: await resolveSku(item.sku),
  quantity: item.qty,
  price_per_unit: (item.priceCents / 100).toFixed(2),
}
```

The local/Stripe price is authoritative for what the customer paid.

No line-item `remote_id` is needed. The Veeqo `sellable_id` is what
actually selects the Veeqo inventory item.

------------------------------------------------------------------------

## 16. Veeqo Order Number

The human-facing Veeqo order number is generated as:

``` ts
const brandName = getBrandBySlug(brandSlug)?.name ?? brandSlug;
const orderNumber =
  `${brandName} - #${orderId.slice(-8).toUpperCase()}`;
```

Examples successfully created:

``` text
BikerShades - #1CD058EB
BikerShades - #AEC1E1B6
```

The complete Supabase UUID remains the canonical local database ID.

The Veeqo `number` field is a human-facing order number, not the Veeqo
internal order ID.

------------------------------------------------------------------------

## 17. Important IDs and References

### `orders.id`

The local Supabase UUID.

### `orders.veeqo_order_id`

The numeric Veeqo order ID returned after successful creation.

### Veeqo `number`

Human-readable order number generated from brand + UUID suffix.

### Stripe Checkout Session ID

Stored locally as `stripe_session_id`.

Useful for identifying the Checkout flow and local webhook idempotency.

### Stripe PaymentIntent ID

Stored locally as `stripe_payment_intent`.

Also sent to Veeqo as:

``` ts
payment_attributes.reference_number
```

This links the Veeqo payment back to the actual Stripe payment
lifecycle.

------------------------------------------------------------------------

## 18. Shipping Cost Decision

The storefront presents shipping to the customer as free.

Therefore:

``` ts
delivery_cost: "0.00"
```

is sent to Veeqo.

Conceptually:

``` text
Products = amount customer paid
Shipping charge to customer = $0
Business later pays fulfillment/postage expense
```

The business's cost of buying a USPS label is not the same as a shipping
charge paid by the customer.

Successful Veeqo orders displayed:

``` text
Shipping cost $0.00
```

as intended.

------------------------------------------------------------------------

## 19. Veeqo Customer vs. Delivery Information

Veeqo separates:

``` text
customer_attributes
```

from:

``` text
deliver_to_attributes
```

because customer/payment information and the actual delivery recipient
are different concepts.

Someone could purchase an item and ship it to somebody else.

Even when they are the same person, the data should remain semantically
separate.

------------------------------------------------------------------------

## 20. Shipping Data from Stripe

Shipping comes from:

``` ts
session.collected_information!.shipping_details!
```

Mapping:

``` text
shipping_details.name                → deliver_to.first_name
shipping_details.address.line1       → address1
shipping_details.address.line2       → address2
shipping_details.address.city        → city
shipping_details.address.state       → state
shipping_details.address.postal_code → zip
shipping_details.address.country     → country
customer email                       → deliver_to.email
customer phone                       → deliver_to.phone
```

The full shipping name is placed into Veeqo `first_name`.

We deliberately do not guess how to split arbitrary human names into
first/last components.

------------------------------------------------------------------------

## 21. Why `name` Was Wrong in `deliver_to_attributes`

The Veeqo create-order shape expects fields such as:

``` json
{
  "first_name": "Frodo",
  "last_name": "Baggins"
}
```

Therefore:

``` ts
name: customer.name
```

was changed to:

``` ts
first_name: customer.name
```

with `last_name` omitted.

Testing confirmed that this works and Veeqo displays the name correctly.

------------------------------------------------------------------------

## 22. Phone Number

A Veeqo fulfillment test showed that the shipping/label workflow
requires a phone number.

The sync therefore checks:

``` ts
if (!customer.phone) {
  throw "Missing shipping phone number";
}
```

The Stripe Checkout contact phone is sent to:

``` ts
customer_attributes.phone
```

and:

``` ts
deliver_to_attributes.phone
```

Stripe Checkout displays the contact phone above the payment section.
Whether the customer checks or unchecks "Billing info is same as
shipping," that contact number remains the number used by this
integration.

------------------------------------------------------------------------

## 23. Billing Address Collection

Originally, billing could be absent because Stripe Checkout does not
necessarily collect a full billing address by default.

Checkout was changed to require it:

``` ts
billing_address_collection: "required"
```

The billing/customer information is then available on:

``` ts
session.customer_details
```

including:

``` text
name
email
phone
address
```

This avoids making an additional Stripe PaymentIntent/Charge API call
solely to retrieve billing information.

------------------------------------------------------------------------

## 24. Stripe Checkout Billing UI Behavior

Checkout provides:

``` text
Billing info is same as shipping
```

If checked, Stripe uses the shipping information for billing.

If unchecked, Stripe exposes separate billing/cardholder fields.

The implementation does not care which option the customer chooses. It
always reads the resulting billing information from
`session.customer_details` and the shipping information from
`shipping_details`.

------------------------------------------------------------------------

## 25. Final Billing and Shipping Mapping

### Stripe billing/customer

``` text
session.customer_details
```

maps to:

``` text
Veeqo customer_attributes
Veeqo customer_attributes.billing_address_attributes
```

### Stripe shipping

``` text
session.collected_information.shipping_details
```

maps to:

``` text
Veeqo deliver_to_attributes
```

This means billing and shipping can be independently different.

------------------------------------------------------------------------

## 26. Veeqo Billing Payload

Veeqo's documented create-order shape includes:

``` json
{
  "customer_attributes": {
    "phone": 7891234567,
    "email": "baggins@ringbearers.org",
    "billing_address_attributes": {
      "first_name": "Frodo",
      "last_name": "Baggins",
      "address1": "The New Bag End",
      "city": "Valinor",
      "state": "The Undying Lands",
      "zip": "VA1 1NB",
      "country": "GB"
    }
  }
}
```

Our implementation follows that shape:

``` ts
customer_attributes: {
  email: customer.email,
  phone: customer.phone,
  ...(customer.billing.address ? {
    billing_address_attributes: {
      first_name: customer.billing.name,
      address1: customer.billing.address.line1,
      ...(customer.billing.address.line2
        ? { address2: customer.billing.address.line2 }
        : {}),
      city: customer.billing.address.city,
      state: customer.billing.address.state,
      zip: customer.billing.address.postal_code,
      country: customer.billing.address.country,
    },
  } : {}),
}
```

Billing information does not need to be saved to our database just for
Veeqo. It can flow from Stripe → webhook → Veeqo.

------------------------------------------------------------------------

## 27. Billing/Shipping Independence Was Tested

A successful test deliberately used different shipping and billing
information.

Veeqo showed:

``` text
SHIPPING
Ansh Ramanath
5945 West Parker Road
Apt 1403
Plano, TX 75093

BILLING
Bob Marly
2111 Rio Grande St
708
Austin, TX 78705
```

This proved that:

-   Stripe billing data is being read correctly.
-   Stripe shipping data is being read correctly.
-   Veeqo receives them through separate fields.
-   We are not copying shipping into billing.

------------------------------------------------------------------------

## 28. Payment Type

The first successful paid Veeqo test used:

``` ts
payment_type: "credit_card"
```

Veeqo's own example also showed:

``` text
paypal
```

so the field is not inherently limited to `credit_card`.

Stripe Checkout can expose multiple payment methods, including card and
other Stripe-supported methods.

Hardcoding `credit_card` could therefore misrepresent a non-card Stripe
payment.

------------------------------------------------------------------------

## 29. `"Stripe"` as Payment Type

We tested:

``` ts
payment_attributes: {
  payment_type: "Stripe",
  reference_number: paymentIntent,
}
```

Veeqo accepted the order.

The Veeqo UI displayed:

``` text
Stripe
```

under Customer Details.

This is a better representation for this integration because Stripe is
the payment processor regardless of which underlying Stripe Checkout
payment method the customer chooses.

The final preferred value is therefore:

``` ts
payment_type: "Stripe"
```

------------------------------------------------------------------------

## 30. Final Veeqo Payload Shape

Conceptually, the final POST is:

``` json
{
  "order": {
    "channel_id": "<resolved channel ID>",
    "delivery_method_id": "<resolved delivery method ID>",
    "number": "BikerShades - #XXXXXXXX",
    "delivery_cost": "0.00",

    "customer_attributes": {
      "email": "<Stripe email>",
      "phone": "<Stripe contact phone>",
      "billing_address_attributes": {
        "first_name": "<billing/cardholder name>",
        "address1": "<billing line 1>",
        "address2": "<optional>",
        "city": "<billing city>",
        "state": "<billing state>",
        "zip": "<billing postal code>",
        "country": "<billing country>"
      }
    },

    "deliver_to_attributes": {
      "first_name": "<shipping recipient>",
      "address1": "<shipping line 1>",
      "address2": "<optional>",
      "city": "<shipping city>",
      "state": "<shipping state>",
      "zip": "<shipping postal code>",
      "country": "<shipping country>",
      "email": "<customer email>",
      "phone": "<customer phone>"
    },

    "line_items_attributes": [
      {
        "sellable_id": "<resolved exact SKU sellable ID>",
        "quantity": 1,
        "price_per_unit": "19.99"
      }
    ],

    "payment_attributes": {
      "payment_type": "Stripe",
      "reference_number": "<Stripe PaymentIntent>"
    }
  }
}
```

------------------------------------------------------------------------

## 31. Successful Multi-Item End-to-End Test

A successful order appeared in Veeqo as:

``` text
BikerShades - #1CD058EB
READY TO SHIP
```

It contained two product lines:

``` text
SP47X96BS-GY-GRSM
quantity 2
$19.99 each

BF8X77KY-PU-GSM-200
quantity 1
$14.99
```

The Veeqo subtotal was:

``` text
$54.97
```

which correctly equals:

``` text
2 × $19.99 + $14.99 = $54.97
```

Veeqo showed:

``` text
Shipping cost: $0.00
Order total: $54.97
```

It also showed inventory allocation from:

``` text
ADDISON OFFICE
```

and displayed available Veeqo inventory for the resolved sellables.

This confirmed SKU resolution, quantities, prices, allocation, customer
details, shipping, billing, and free-shipping representation.

------------------------------------------------------------------------

## 32. Successful Stripe + Separate Billing Test

Another successful Veeqo order appeared as:

``` text
BikerShades - #AEC1E1B6
READY TO SHIP
```

The Veeqo Customer Details area displayed:

``` text
Stripe
```

as the payment type.

The test deliberately used different billing and shipping addresses, and
both appeared in their correct columns.

This confirmed two important final changes simultaneously:

1.  `"Stripe"` is accepted and displayed as the Veeqo payment type.
2.  Billing information truly comes from Stripe billing/customer details
    rather than shipping.

------------------------------------------------------------------------

## 33. Refund Handling Already in the Webhook

The same webhook also handles:

``` text
charge.refunded
```

It extracts the PaymentIntent and updates the matching order:

``` ts
refunded_cents: charge.amount_refunded
```

For a full refund:

``` ts
status: "refunded"
```

is also applied.

If no normal order matches the PaymentIntent, the webhook attempts to
update the corresponding TBYB submission.

This is separate from Veeqo order creation but remains part of the
overall Stripe backend workflow.

------------------------------------------------------------------------

## 34. TBYB vs. Normal Order Checkout

Stripe Checkout metadata distinguishes workflows.

For:

``` text
metadata.type = "tbyb"
```

the webhook updates the TBYB submission and stores Stripe/shipping
information.

For:

``` text
metadata.type = "order"
```

the webhook:

``` text
creates/upserts local order
creates/upserts items
syncs to Veeqo
```

This keeps both flows inside the same signature-verified webhook while
making their behavior explicit.

------------------------------------------------------------------------

## 35. Things We Intentionally Did Not Add

### No hardcoded Veeqo channel ID

We resolve the desired active direct channel by configured name.

### No hardcoded delivery-method ID

We resolve the desired method by configured name.

### No line-item remote ID

The Veeqo sellable ID is what matters for inventory allocation.

### No arbitrary duplicate-SKU choice

A true exact-SKU ambiguity is an error.

### No fake billing information

Stripe's actual billing data is used.

### No customer shipping charge

The site advertises free shipping, so Veeqo receives `0.00`.

### No fire-and-forget sync

The webhook awaits Veeqo work.

### No blind retry of uncertain remote creation

A stuck `"Sync in progress"` is investigated rather than risking a
duplicate order.

### No large sync-status/error framework

`veeqo_order_id` + `veeqo_error` are enough for the current operational
model.

------------------------------------------------------------------------

## 36. Main Lessons

### Local and remote idempotency are different

A unique Stripe Session can prevent duplicate local orders but cannot by
itself prevent two concurrent Veeqo POSTs.

### Atomic check-and-set solves the race

Do not:

``` text
SELECT
check
UPDATE
```

Do:

``` text
UPDATE ... WHERE state is unclaimed
RETURNING id
```

### Remote side effects can succeed without local acknowledgement

Veeqo may create an order and the backend may die before Supabase
records the ID. That is why retry behavior must be conservative.

### Serverless work that matters should be awaited

Do not rely on an invocation remaining alive after returning its
response.

### Search results are not identities

`GET /products?query=sku` is only discovery. Exact `sku_code` matching
is required.

### Channel mappings are not separate inventory items

Multiple channels can reference one underlying Veeqo sellable.

### Billing, customer, and shipping are distinct concepts

Model them separately even when they usually contain the same
person/address.

### Processor and payment method are distinct

`Stripe` is a useful Veeqo payment label even when Stripe itself
processes card, Cash App, Klarna, bank, or other payment methods.

### Dynamic resource resolution is more resilient than fragile IDs

The integration describes the channel/method it wants and resolves the
current Veeqo IDs.

------------------------------------------------------------------------

## 37. Final Architecture

``` text
                            STRIPE
                               │
                  checkout.session.completed
                               │
                               ▼
                       Next.js webhook
                               │
                    verify Stripe signature
                               │
                               ▼
                           SUPABASE
                     upsert local order
                     upsert order items
                               │
                               ▼
                    atomic Veeqo claim
            NULL error → "Sync in progress"
                               │
                         claim won?
                         /        \
                       no          yes
                       │            │
                     stop           ▼
                            resolve resources
                    ┌──────────┬──────────┐
                    ▼          ▼          ▼
                 channel    delivery     SKUs
                              method   → sellable IDs
                    └──────────┴──────────┘
                               │
                               ▼
                        VEEQO POST /orders
                               │
                    ┌──────────┴──────────┐
                    ▼                     ▼
                 success                failure
                    │                     │
          save veeqo_order_id      save veeqo_error
          clear temporary error           │
                    └──────────┬──────────┘
                               ▼
                     webhook returns 200
```

The resulting system now has a tested Stripe → Supabase → Veeqo pipeline
with concurrency protection, correct inventory sellable resolution, real
billing and shipping separation, free-shipping representation, Stripe
payment labeling, and clear operational state for success or failure.
