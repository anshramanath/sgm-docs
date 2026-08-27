# Session Notes (August 23, 2026)

## DB Schema Changes

### Timestamps
Added `created_at` / `updated_at` to tables that were missing them:
- `categories`, `products` — added both (001)
- `orders` — added `updated_at` (003)
- `variations`, `tbyb_packages`, `tbyb_submissions`, `prescription_frames`, `rx_orders` — already had both

### RLS
Enabled row level security on all catalog tables in 001:
`brands`, `categories`, `products`, `product_categories`, `variations`, `product_images`, `variation_images`, `description_images`, `product_description_images`, `admins`

**Important**: These tables have no SELECT policies — anon/authenticated reads are blocked. All admin reads use service role so the admin panel is unaffected. If the frontend ever queries these tables directly with the anon key, it would get empty results.

### updated_at Triggers
Rather than manually passing `updated_at` in every server update call, we use a single DB trigger function that fires automatically on every UPDATE.

**Function** (defined in 001, used by all migrations):
```sql
create or replace function set_updated_at()
returns trigger language plpgsql as $$
begin
  new.updated_at = now();
  return new;
end;
$$;
```

Triggers added for: `categories`, `products`, `variations` (001), `orders` (003), `tbyb_packages`, `tbyb_submissions` (004), `prescription_frames`, `rx_orders` (005).

`drop_schema.sql` updated to include `drop function if exists set_updated_at()`. Triggers are dropped automatically via CASCADE when their tables are dropped.

### initial_schema.sql
Kept in sync with 001–005. Run a diff anytime to verify:
```bash
cat 001 002 003 004 005 > /tmp/combined.sql && diff combined.sql initial_schema.sql
```
Only differences should be blank lines between section headers.

---

## How Postgres Triggers Work

- `BEFORE UPDATE` means: after your update has been applied to `NEW` in memory, but before it's committed to disk — run the trigger.
- `NEW` = OLD row + your changes, in memory. The trigger can modify it. Whatever it returns is what gets written.
- `OLD` = the row on disk before the update. Used to compare what changed (e.g. the `decrement_total_sales_on_refund` trigger checks `old.refunded_cents > 0` to prevent double-decrementing).
- `FOR EACH ROW` fires once per affected row with its own `OLD`/`NEW`. `FOR EACH STATEMENT` fires once for the whole statement with no row access — useful for audit logging, rarely used for data modification.
- The trigger is **atomic** with the UPDATE — if either fails, both roll back.

---

## Stripe Webhook

### Idempotency
All update paths return 500 on failure (Stripe retries) and 200 on success or intentional no-ops. Idempotency is ensured by:
- `orders` upsert with `onConflict: "stripe_session_id"`
- `order_items` upsert with `onConflict: "order_id,sku"`
- `decrement_total_sales_on_refund` trigger guards against double-decrement via `old.refunded_cents > 0`
- Veeqo sync uses an atomic claim (`veeqo_error = "Sync in progress"` with `.is("veeqo_order_id", null).is("veeqo_error", null)`) so only one invocation proceeds

### Events
Only two events needed:
- `checkout.session.completed` — handles orders, tbyb submissions, rx orders
- `charge.refunded` — handles refunds for all three

### Going Live
When moving from Stripe test to live mode:
1. Create a new webhook endpoint in the live mode dashboard
2. Select the same two events
3. Update `STRIPE_WEBHOOK_SECRET` in Vercel to the new live signing secret
4. Update `STRIPE_SECRET_KEY` in Vercel to the `sk_live_...` key

---

## Products Page Feature Flag

Products removed from the sidebar NAV in `sidebar.tsx` for the time being — the page and all its code still exist, just not linked. Re-add the entry to NAV when ready:
```ts
{ label: "Products", path: "/products" },
```

---

## cart_items / bookmarks

Dropped `updated_at` and `created_at` from these tables — they're transient user data and timestamps add no value. The migration (002) and server code (cart route) were updated accordingly.

**Key lesson**: If you drop a column from the DB before removing it from server code, inserts will fail during the deployment window. Always deploy the code change first, then drop the column.
