# Project Learnings (August 23, 2026)

Consolidated understanding of the bikershades-migration project — architecture, conventions, and lessons from building/operating it. Written as a knowledge-transfer doc, not a session log (see `MIGRATION_SESSION_SUMMARY.md` for the blow-by-blow of the most recent session, `SGM_DATA_README.md` for the backup folder itself).

## What this project is

Migrates three WooCommerce storefronts (`prosport` / proSPORT Sunglasses, `monster` / Sunglass Monster, `bikershades` / BikerShades) into a single Supabase-backed multi-brand platform. Each brand keeps its own WooCommerce site as the original data source but shares one Postgres schema and one set of Supabase Storage buckets (one bucket per brand). Two adjacent features live in the same DB: `prescription-frames` (a curated frame catalogue, separate from the WooCommerce pipeline) and TBYB — Try Before You Buy — packages.

Active code lives in `final-migration/`. `old-migration/` and `new-migration/` are abandoned — ignore them entirely.

## Pipeline architecture

18 sequential scripts in `final-migration/pipeline/`, each reading the previous stage's output and writing its own to `data/{brand}/{stage}/items.json` (gitignored). Run via `BRAND={brand} python3 pipeline/{script}.py`. Full detail in `PIPELINE.md`; the shape that matters most:

```
fetch_products → create_items → slim_items → reshape_items_1 → reshape_items_2
  → validate_items → remove_items → merge_attributes → dedup_images
  → generate_content → (MCP server writes descriptions) → categorize_items
  → normalize_image_names → decode_entities → validate_items_final
  → download_items → upload_items → import_items
```

Each stage does one job and hands off a clean JSON file — nothing reaches into a later stage's territory. `validate_items_final.py` is the last gate before anything touches images or the DB; everything after it should be able to trust the data completely.

**Validation philosophy**: checks accumulate — a product collects *all* its failures before being flagged, one failure is enough to exclude it entirely. Flagged items go to a `flagged.json` alongside the clean `items.json`; nothing flagged ever proceeds downstream.

**Variable vs. simple products** is the core branch point everywhere: a variable product has `sku: null` at the product level and ≥2 variations; a simple product has a real `sku` and an empty `variations` array. Checks apply at the right level accordingly — Veeqo/SKU-structure checks are variation-level for variable products, product-level for simple ones.

## Database conventions

- Pipeline JSON fields are camelCase; DB columns are snake_case. The mapping only happens in `import_items.py` — nowhere else needs to know about it.
- **Two separate slug fields, never conflate them**: `slug` (pipeline key / data directory name / storage bucket path — `prosport`/`monster`/`bikershades`) vs. `brand_slug` (DB `brands.slug` / Supabase bucket name — `prosport-sunglasses`/`sunglass-monster`/`bikershades`).
- **Deferred fields** — `description`, `summary`, `categories` — are left null/empty through most of the pipeline and only populated in later stages (AI content generation, brand-specific categorization). Early-stage validators explicitly skip them; only `validate_items_final.py` checks them, since by then they're populated.
- Every entity table has a surrogate `uuid` PK; join tables use a composite PK of their two FK columns (enforces uniqueness, gives a free index, keeps Supabase's PostgREST/Realtime tooling happy — all of which lean on a PK existing).
- `created_at`/`updated_at` exist only on tables with an actual admin-editing story (`categories`, `products`, `variations`, `orders`, `tbyb_packages`, `tbyb_submissions`, `prescription_frames`, `rx_orders`) — not on pure join tables, static image tables, or ephemeral per-user tables (`cart_items`, `bookmarks`) that get replaced/deleted rather than tracked.
- `updated_at` is maintained by a shared `set_updated_at()` trigger (`before update`, sets `now()`), not app-managed — but it fires on *any* `UPDATE` to the row, including the view-count increment functions. So on `categories`/`products`, `updated_at` reflects "last viewed or edited," not purely "last edited" — worth remembering if it's ever used to mean "content changed."
- **RLS**: catalog tables (`brands` through `admins`) are locked down with RLS enabled and *zero policies* — full deny for `anon`/`authenticated`. This is safe specifically because nothing in the frontend reads these tables directly with the anon key; everything goes through admin/server code using a direct Postgres connection (which bypasses RLS as the table owner) or `service_role`. If that ever changes — if the anon key starts reading catalog data directly — these tables would need an explicit `for select using (true)` policy added, or reads would silently break.
- Import is deliberately **destructive per brand**: every run deletes all of that brand's data (explicit ordered deletes, leaf to root — not relying on `ON DELETE CASCADE` alone, even though the schema has it for correctness on fresh deploys) and reinserts everything fresh with brand-new UUIDs. This means re-running import for a brand that has live users is not a no-op: any `cart_items`/`bookmarks` row referencing an old product UUID gets cascade-deleted permanently when that product is wiped, and `view_count`/`total_sales` reset to the import's defaults rather than being preserved. Fine during initial setup; a real hazard once the site has live traffic.
- Import has **no fallbacks** — every field access on `p["..."]` is direct, no `.get()`. A `KeyError` during import means a validation gap upstream, not something to handle defensively in the import script itself.

## Image handling

- Filenames are a **deterministic pure function of the source URL** (`make_filename()` — strip everything before `/uploads/` or `/gallery/`, replace `/` with `_`, handle query strings). This one property is what made same-day disaster recovery possible: given only `items.json` and knowledge of this function, every image's expected filename and Supabase public URL can be recomputed from scratch, with no dependency on the original source site or any previously-saved map file.
- Three files chain together to resolve an image at import time: `download_map.json` (source URL → local path), `upload_map.json` (local path → Supabase public URL), and `resolve_src()` in `import_items.py` walks both. Both are crash-safe/resumable by design — but that resumability cuts both ways: if the *target* (local disk, or a Supabase bucket) gets wiped without also clearing these local map files, the scripts will trust the stale map and silently skip re-doing the work, reporting false success. Always verify against the actual target state after a wipe, not just the map's own claims.
- Supabase Storage never deletes anything on its own and never re-encodes/transcodes on upload or download — it's a pure byte store. Combined with deterministic filenames, this means storage is a durable, independent backup of every image the pipeline has ever successfully uploaded, for as long as the bucket exists.
- A "successfully imported" product doesn't mean every one of its images made it in — `download_items.py`'s own coverage check only guarantees *at least one* `productImages` entry resolved per item, not full coverage. Individual missing images (a secondary angle shot, a sizing diagram) fail silently as warnings during import, not as hard errors.

## Operational lessons

- **Transaction commit boundaries are the real unit of atomicity, and they don't always line up with intuition.** `import_items.py` commits after the wipe, after the brand insert, then every 50 products. A crash anywhere rolls back everything since the last commit — including products that individually looked "done." The brand row surviving while zero products do (because it has its own earlier standalone commit) is a direct, provable consequence of this, not a bug.
- **A script's self-reported success is not verification.** Trusting `upload_map.json`'s claim of "already uploaded" after the actual bucket had been deleted produced a `0 uploaded, N skipped` run that looked clean but did nothing. The fix pattern that actually catches this: query the live target directly (list the real bucket contents, query the real DB row counts) and diff against what's expected from the source data — never just trust a script's own summary line or a local cache file.
- **A destructive action's blast radius is often larger than the immediate task.** Auditing "did we back up the migration images" before a full storage wipe surfaced `rx/`/`tbyb/` folders that had nothing to do with the pipeline at all — live customer-order upload folders sitting in the same bucket. Worth explicitly listing what's actually *in* a system before deleting it, not just what you expect to be there.
- **Source-of-truth external systems can silently degrade.** All three original WooCommerce sites became partially unreachable (broken SSL cert on one, 403s on two) between when data was originally scraped and when it was needed again — with no warning, no changelog, nothing in this repo predicting it. Anything that depends on a third party staying reachable indefinitely should have a durable, independent copy the moment it's fetched (which is exactly what Supabase Storage ended up providing here, incidentally rather than by original design).

## What's been built / improved

- Four completeness checks added to `validate_items_final.py`: duplicate option slugs within an attribute, duplicate product names, duplicate attribute names, and `variationImages` sort/dup checks (bringing it in line with `productImages`/`descriptionImages`) — plus a simple-product min/max price equality check.
- One check added to `prescription-frames/validate.py`: duplicate color slugs within a single frame.
- Schema: `created_at`/`updated_at` + RLS lockdown across all catalog tables, `set_updated_at` trigger infrastructure, `initial_schema.sql` restored as an accurate concatenation of the numbered migration files.
- Fixed `prescription-frames/migrate.py`, which was silently unrunnable (wrong env file path, wrong env var name).
