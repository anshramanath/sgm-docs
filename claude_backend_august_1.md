# DEVLOG — TBYB Backend + Security (August 1, 2026)

## What Was Built

### Three TBYB Endpoints

**`GET /api/public/packages`**
- Returns all packages for a brand via `adminSupabase`
- No auth required — packages are public catalog data

**`POST /api/user/upload`**
- Authenticated — validates JWT via user client, but uploads via `adminSupabase`
- Sanitizes filename, uploads to `{brandSlug}/tbyb/{name}-{uuid}`
- Returns public URL for use in form submission body

**`POST /api/user/tbyb`**
- Authenticated — validates JWT, extracts `client.user.id`
- Looks up package via `adminSupabase` (no select policy for authenticated on `tbyb_packages`)
- Inserts submission via `adminSupabase` — snapshot pattern, all fields NOT NULL
- Returns `{ id: uuid }`

### Database — `004_tbyb.sql`

`tbyb_packages` — catalog of TBYB packages per brand. Admin-only writes.

`tbyb_submissions` — one row per form submission. Stores a full snapshot of the package at time of submission (name, slug, price, image, pairs, brands) so submissions survive package edits or deletions.

All 28 submission columns are `NOT NULL`. Frontend always sends every field; optional fields use `"None"` string, never `null`. If anything is missing the insert fails atomically — no partial rows.

---

## Security Decisions

### Grants + RLS are two separate gates

- **Grant** — can this role touch the table at all?
- **RLS policy** — which rows can they access?
- Both must pass. Admin client bypasses both.

### Why authenticated gets no insert on `tbyb_submissions`

The anon key is public — anyone can see it in the frontend bundle. If authenticated had insert privileges, someone with a valid JWT could skip the route entirely and insert a submission with a fake `package_price_cents`. By removing the insert grant, the only path to insert is through the route, which pulls package details from the DB itself via admin client.

Same logic applies to `orders` and `order_items` — no insert grant, webhook uses admin client.

### Why authenticated has insert on `cart_items` and `bookmarks`

No price or integrity constraint to protect — it's user preference data. Spam inserts only pollute the attacker's own cart. RLS ensures `auth.uid() = user_id` so cross-user access is impossible.

### Why upload uses `adminSupabase` not user client

Storage RLS is on `storage.objects`. No insert policy exists for authenticated users, so the user client would be denied. Admin client bypasses storage RLS. The route is the gatekeeper — it validates auth, sanitizes the filename, and scopes the path to `tbyb/`.

Could add a storage policy to allow authenticated uploads, but that's more privilege than needed — the route already enforces everything.

### `USING` vs `WITH CHECK` in RLS policies

- `USING` — row filter applied before the operation (SELECT, UPDATE, DELETE). "Which rows can you touch?"
- `WITH CHECK` — validates the new row state after a write (INSERT, UPDATE). "Is what you're writing valid?"
- UPDATE uses both: `USING` to find the row, `WITH CHECK` to ensure the updated row still satisfies the condition (prevents changing `user_id` to someone else's)

### `order_items` policy joins through `orders`

`order_items` has no `user_id` column, so ownership is verified via the parent:

```sql
using (exists (
  select 1 from orders
  where orders.id = order_items.order_id
    and orders.user_id = auth.uid()
))
```

Both conditions required: item belongs to the order AND you own the order. The `exists` returns true/false — you can see the item only if the parent order is yours.

### Authenticated user surface area

The worst an authenticated user can do is spam inserts into their own cart/bookmarks. No cross-user data access, no writes to orders, submissions, products, or packages.

---

## Supabase Patterns

### Admin vs user client

```ts
const client = await createUserClient(req);   // validates JWT, bound by grants + RLS
if (!client) return err("Unauthorized", 401);

const adminSupabase = createAdminClient();     // bypasses grants + RLS entirely
const { supabase } = client;                  // use when you want RLS enforced
```

- Use user client when RLS should enforce row-level access (reading user's own data)
- Use admin client when the route is the authority (writes with business logic, lookups with no select policy)

### Snapshot pattern

Instead of storing a FK to `tbyb_packages`, the submission stores a copy of the package data at insert time. Same pattern as `order_items` (stores product name, price, image — not a FK to products).

Why: packages can change or be deleted. The submission record is a receipt — it should reflect what the user actually submitted against, not the current state of the package.

### NOT NULL as validation

All submission fields are NOT NULL in the DB. No `?? null` fallbacks in the route. If anything is missing, the insert fails atomically — the entire transaction is rolled back, nothing is written, a 500 is returned. Since missing fields are either a frontend bug or a malicious request, this is the right behavior.

### Storage upload path

`{brandSlug}/tbyb/{sanitizedName}-{uuid}`

- Filename sanitized: `file.name.replace(/[^a-zA-Z0-9._-]/g, "_")`
- UUID suffix prevents collisions
- `tbyb/` folder scopes uploads away from product/package images

---

## Schema Hygiene

`initial_schema.sql` — full agglomeration of 001–004. Run this on a fresh DB to get the complete schema.

`drop_schema.sql` — drops all tables then functions. Tables must go first because triggers depend on functions — dropping the tables removes the triggers, then the function drops succeed cleanly.
