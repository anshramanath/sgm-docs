# Learnings & Build Log (June 18, 2026)

## What Was Built

### Stripe Checkout Integration

**Endpoints:**
- `POST /api/user/checkout` — creates a Stripe checkout session, returns a URL
- `POST /api/webhooks/stripe` — handles `checkout.session.completed`, creates order + order_items
- `POST /api/public/validate-cart` — checks if cart item SKUs still exist in the catalog

**Schema additions:**
- `orders` — one row per completed payment
- `order_items` — snapshot of what was purchased (product_slug, sku, name, image_src, price_cents, quantity, attribute)

---

## Key Design Decisions

### Orders are created only in the webhook, not at session creation
No pending order rows. The webhook is the authoritative signal that payment happened. Simpler, no DB state to manage during checkout.

### Idempotency key = `userId:hash(cart):orderCount`
Same cart + same user + same order count = same Stripe session returned. Prevents duplicate sessions on double-click. Including order count means a repurchase of the exact same cart (after clearing and refilling) still creates a new session.

### Cart hash
SHA-256 of the sorted cart items. Sorted by SKU so insertion order doesn't affect the hash. Fixed-length output regardless of cart size.

### No stock gating at checkout
Stock is stored but never checked programmatically during checkout. Orders are placed regardless. If something can't be fulfilled, the owner refunds manually via Stripe. Validates only that the product/variation *exists*, not that stock is available.

### Validate-cart uses SKU not product slug + attribute
SKU uniquely identifies a variation. No need for attribute matching logic. Two parallel queries — one against `variations` (joined to products for brand scoping), one against `products` (for simple products with no variations). Result is a set of found SKUs, cross-referenced against the input.

### Order history is a snapshot
`order_items` stores name, image, price, sku, attribute at purchase time. No FK to products. If a product is deleted, order history is unaffected.

### User deletion preserves orders
`orders.user_id` is nullable with `on delete set null`. Deleting a user nulls out their user_id on past orders rather than cascading. Order records are financial data worth retaining.

### Image passed through Stripe, not looked up in webhook
`product_data.images` is set on the line item at session creation. The webhook expands `data.price.product` via `listLineItems` to retrieve the image, avoiding a separate DB query.

### SKU extracted from line item description
Description format: `"Product Name (SKU-123)"`. SKU extracted with `lastIndexOf("(")` — more robust than regex since it always finds the last `(`, so product names containing parentheses don't break it.

### Name extracted symmetrically
`desc.slice(0, start - 1)` — strips from the last `(` backwards. Consistent with SKU extraction.

---

## RLS & Auth Design

| Table | RLS | Grant | Reason |
|-------|-----|-------|--------|
| `brands`, `categories`, `products`, etc. | yes | none | Admin client only, no user access |
| `cart_items`, `bookmarks` | yes | select/insert/update/delete | Users manage their own rows directly |
| `orders`, `order_items` | yes | select only | Users read, webhook (admin) writes |
| `admins` | yes | none | Admin client only |

### Why cart/bookmarks allow direct user mutation
The anon key is public but requires a valid Supabase JWT to do anything. RLS scopes all operations to `auth.uid() = user_id`. A user can only touch their own rows — directly calling Supabase is equivalent to going through the app.

### Why orders are select-only for authenticated role
Users cannot insert or update orders even with a valid JWT. Write path is exclusively the webhook using the service role, which bypasses RLS entirely. Prevents forged orders.

### createUserClient
Validates the Bearer token internally via `getUser()` (network call to Supabase). Returns `{ supabase, user }` or null. Routes destructure what they need — `user.id` for inserts, `supabase` for queries.

---

## Stripe Concepts

### Idempotency key
Passed as a header to Stripe API calls. Stripe caches the response for 24h. Same key + same parameters = same response returned, no new object created. Key must match parameters exactly — different parameters with the same key returns an error.

### checkout.session.completed
The authoritative payment event. Fires when Stripe confirms payment. `session.client_reference_id` carries the user ID. `session.metadata` carries custom fields (brandSlug). `session.payment_intent` is the payment intent ID needed for refunds.

### stripe_session_id vs stripe_payment_intent
- `stripe_session_id` — identifies the checkout session. Used for idempotency check in webhook and to retrieve session details from Stripe later.
- `stripe_payment_intent` — identifies the payment transaction. Used for issuing refunds.
- Both are `not null unique` on the orders table — double layer of duplicate protection.

### Webhook idempotency
Stripe can deliver the same event multiple times. Before inserting an order, check if `stripe_session_id` already exists. If it does, return 200 and skip. The unique constraint on `stripe_session_id` also catches race conditions where two deliveries slip through simultaneously — the second insert fails at the DB level.

### Race conditions
Two concurrent webhook deliveries for the same event: both pass the idempotency check, both try to insert. The unique constraint on `stripe_session_id` blocks the second insert. The second request returns 500 which is fine — Stripe doesn't retry successful deliveries.

### Payment method selection
Bank redirects (Bancontact, BLIK, EPS) and buy now pay later (Affirm, Klarna) disabled. Remaining: cards, wallets (Apple Pay, Google Pay, etc.), Pix. All immediate — `stripe_payment_intent` is always present when `checkout.session.completed` fires, so it's `not null`.

### customer_email
Passed to `stripe.checkout.sessions.create` to pre-fill the email field on Stripe's hosted checkout page. Fetched from `supabase.auth.getUser()`.

---

## DB Migrations

| File | Status | Purpose |
|------|--------|---------|
| `001_initial_schema.sql` | Source of truth for fresh installs | Full schema |
| `002_user_cart_bookmarks.sql` | Historical | Cart and bookmarks initial migration |
| `003_orders.sql` | Applied to live | Orders and order_items tables |
| `004_orders_updates.sql` | Apply to live, then delete | user_id nullable + stripe_payment_intent unique |
| `drop_schema.sql` | Dev only | Wipes all tables |

---

## API Changes Since Last Session

- `cart_items` and cart API now include `sku`
- `validate-cart` endpoint added (SKU-based existence check)
- `checkout` endpoint added
- `webhook` endpoint added
- `in_stock` filter removed from products, sale, and search endpoints — stock is not gating anything
- `createUserClient` is now async, validates token internally, returns `{ supabase, user }`
