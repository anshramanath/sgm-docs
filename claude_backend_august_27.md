# Session Notes (August 27, 2026)

## What was built

### Atomic product save RPC (`006_save_product.sql`)

The entire `saveProduct` flow — product row, categories, images, description images, variations, variation images — now runs in a single Postgres transaction via `supabase.rpc("save_product", {...})`. All or nothing. A failure at any point rolls back everything.

**Why this matters**: the old TypeScript approach made ~15 individual DB calls. A crash midway left partial state. The RPC is atomic by nature — Postgres guarantees the whole thing commits or nothing does.

### Unique constraints

Added `unique (product_id, src)` to `product_images` and `unique (variation_id, src)` to `variation_images`. These enable `ON CONFLICT` upserts, collapsing insert + update-sort-order into a single statement per image table.

**Applied via**: `ALTER TABLE` at the top of `006_save_product.sql`. Pre-checked with duplicate queries before applying.

---

## How the RPC works

### Parameters
All snake_case, prefixed `p_` to avoid ambiguity with column names. `p_summary` is `text[]` (not `jsonb` — matches the DB column type). `p_attributes`, `p_images`, `p_description_images`, `p_variations` are `jsonb`.

### Product row
- `p_is_new = true` → INSERT
- `p_is_new = false` → UPDATE
- `total_sales` handled by CASE + COALESCE:
  - Simple product: `coalesce(total_sales, 0)` — preserves existing value, defaults to 0 if switching from variable (where total_sales was null)
  - Variable product: forced to null

### Categories
Insert new with `ON CONFLICT DO NOTHING`, delete removed with `NOT (category_id = ANY(p_category_ids))`. Order doesn't matter — insert and delete operate on disjoint sets.

### Product images / Variation images
Upsert on `(product_id, src)` / `(variation_id, src)` — inserts new, updates sort_order for reordered. Delete where src not in input.

### Description images
Shared table — never deleted, only upserted. Junctions (`product_description_images`) are diffed: insert new links, delete removed ones. Scoping by `product_id` alone is sufficient since product_id is globally unique.

### Variations
1. Delete removed: `id NOT IN (non-null ids from input)` — null ids = new variations, excluded from keep list
2. Loop: insert new (null id → `RETURNING id`), update existing (non-null id)
3. Variation images: upserted + deleted per variation, inside the loop

**Simple product case**: `p_variations = []` → the delete removes all variations, the loop runs 0 times. Variation images cleaned up via `ON DELETE CASCADE`.

---

## SQL concepts covered

**`DECLARE`** — function-scoped local variables. Every call gets fresh copies.

**`BEGIN...END`** — marks the function body. Also used for transaction blocks.

**`$$`** — dollar-quoting delimiter for function bodies. Avoids escaping single quotes.

**`RETURNS VOID`** — function has no return value. Other options: `RETURNS int`, `RETURNS uuid`, `RETURNS SETOF table_name`, `RETURNS TABLE(col type, ...)`.

**`JSONB`** — opaque blob until you operate on it. `->` extracts as JSON (keeps structure), `->>` extracts as text. `jsonb_array_elements(arr)` expands a JSON array into rows with a `value` column — makes it iterable.

**`INSERT ... SELECT`** — used instead of `INSERT ... VALUES` when the data comes from a set-returning operation (unnest, jsonb_array_elements, a join). `VALUES` is for fixed literals.

**`ON CONFLICT`** — requires a unique constraint or PK to detect the conflict. `DO NOTHING` skips, `DO UPDATE SET col = excluded.col` updates the conflicting row. `excluded` refers to the row that was attempted — like `NEW` in a trigger.

**`NOT IN (subquery)`** vs **`NOT (id = ANY(array(...)))`** — equivalent. `NOT IN` with a subquery is cleaner. `ANY` requires an array.

**`USING` in DELETE** — brings another table into scope for filtering, like a join. You specify the join condition manually in `WHERE` — no auto-detection from FK.

**`GROUP BY` + `HAVING`** — `GROUP BY` groups rows, aggregate functions (`COUNT(*)`, `SUM`, etc.) compute per group. `HAVING` filters on aggregated results (like `WHERE` but for groups). `SELECT COUNT(*)` and `HAVING COUNT(*) > 1` are independent — you can use aggregates in `HAVING` without putting them in `SELECT`.

**`RETURNING id INTO v_var_id`** — captures a value from a DML statement into a local variable. Used after inserting a new variation to get its generated UUID.

**`COALESCE(val, default)`** — returns the first non-null argument. `coalesce(null, 0) = 0`.

**`UNNEST(array)`** — expands an array into rows. `select unnest('{a,b,c}'::text[])` returns 3 rows.

**PL/pgSQL FOR loop** — `for v_var in select value from jsonb_array_elements(p_variations)` is the SQL equivalent of `for (const v of variations)`.

**Scalar** — a single value, as opposed to a set/array. `v_var_id` is a scalar UUID, reset each loop iteration.

**`NOT IN` with empty subquery** — `x NOT IN (empty set)` = TRUE for all rows. This is how `p_variations = []` causes all variations to be deleted.

**Transactions within a function** — all statements in a PL/pgSQL function run in one implicit transaction. Mutations made earlier in the function are visible to later statements in the same call. Postgres evaluates the full WHERE against all matching rows at once, not literally one at a time — though the row-by-row mental model is valid for reasoning.

---

## Deployment order

Always: DB migration first, then code deploy. The code calls `save_product` — if the function doesn't exist yet, every save will fail. The constraint alters are also required before the function since the `ON CONFLICT` clauses depend on them.

---

## Branch strategy

Feature branch → test on staging → merge to main → deploy. Main stays clean and deployable at all times. GitHub branch protection on main prevents accidental deletion or force-push.

Remote branch delete (GitHub) and local branch delete (`git branch -d`) are independent operations.

---

## Stripe CLI

`stripe listen --forward-to localhost:3000/api/webhooks/stripe` intercepts events and forwards to local — completely separate from the dashboard webhook. The CLI has its own auth session (`stripe login`), independent of the API key in `.env.local`. Use the `whsec_` the CLI prints as `STRIPE_WEBHOOK_SECRET` locally.
