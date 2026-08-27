# Migration Session Summary (August 23, 2026)

## Schema changes

**`final-migration/db/001_core_catalog.sql`**
- Added `created_at`/`updated_at` (`timestamptz not null default now()`) to `categories`, `products`, `variations`
- Enabled RLS on all 10 catalog tables (`brands`, `categories`, `products`, `product_categories`, `variations`, `product_images`, `variation_images`, `description_images`, `product_description_images`, `admins`) — no policies, so `anon`/`authenticated` are fully locked out while `service_role`/direct `postgres` connections (which bypass RLS) are unaffected. Confirmed safe because `increment_category_view`/`increment_product_view` are only ever called from admin/server code, not the public frontend.
- Added `set_updated_at()` trigger function and attached `before update` triggers to every table with an `updated_at` column: `categories`, `products`, `variations` (001), `orders` (003), `tbyb_packages`, `tbyb_submissions` (004), `prescription_frames`, `rx_orders` (005) — 8 total.
- Note: this trigger fires on *any* `UPDATE`, including `increment_category_view`/`increment_product_view`'s view-count bumps — so `updated_at` reflects "last viewed or edited," not purely "last edited."

**`final-migration/db/003_orders.sql`**
- Added `updated_at` to `orders` (already had `created_at`).

**`final-migration/db/initial_schema.sql`**
- Regenerated as an exact concatenation of `001`–`005` in order (was stale — predated the split into numbered files, missing RLS/timestamps/the tbyb seed data).

**`final-migration/db/drop_schema.sql`**
- Verified complete — drops every table and function across `001`–`005`, including `set_updated_at()`, in valid dependency order.

## Validator improvements

**`final-migration/pipeline/validate_items_final.py`** — 4 new checks added, confirmed no regressions on any brand's current data:
1. Duplicate option slugs within a single attribute (e.g. two `color` options both slugging to `blue`) — frontend keys options by slug, so a collision breaks rendering and variation selection.
2. Duplicate product `name` across the catalogue (mirrors the existing duplicate-`slug` check) — DB has `unique(brand_slug, name)`, so without this check a collision would crash `import_items.py` mid-run.
3. Duplicate attribute *names* within one product's `attributes` array — previously silently overwrote in the validator's internal dict, masking the issue.
4. `variationImages` now gets the same sort-order-uniqueness and duplicate-`src` checks that `productImages`/`descriptionImages` already had (previously inconsistent).
5. Simple products must have `minPriceCents == maxPriceCents`.

**`prescription-frames/validate.py`**
- Added a per-frame check that color slugs are unique within that frame's `colors` array (scoped per frame, not global).

## The incident: accidental local data loss + recovery

- Deleted `data/{brand}/download-items/` and `data/{brand}/upload-items/` for all 3 brands (~124MB) to force a clean re-download/re-upload verification pass.
- Discovered the original source sites are now largely unreachable: `prosportsunglasses.com` and `bikershades.com` return `403` even via plain `curl`; `sunglassmonster.com` has a broken SSL cert (hostname mismatch) — none of these are issues on our end. A fresh download attempt got 0/355 (prosport), 1/706 (monster), 0/2255 (bikershades) images.
- Recovery: since filenames are a deterministic pure function of the source URL (`make_filename()` in `download_items.py`) and nothing in the pipeline ever deletes from Supabase Storage, wrote a one-off script to recompute expected filenames from `items.json` and re-download every image directly from Supabase Storage instead of the broken source sites — bit-for-bit identical bytes (Storage never re-encodes/transcodes; the pipeline never processes images either).
- Result: recovered exactly what was there before — 355/355 prosport, 705/706 monster, 2224/2255 bikershades. The shortfalls are pre-existing, permanent gaps (1 SSL failure, 31 403s) that were never successfully uploaded in the first place, confirmed via direct Supabase Storage `404`/`NoSuchKey` checks and a full bucket-content search.
- Verified thoroughly: file counts on disk match `download_map.json`/`upload_map.json` entries match bucket contents (checked via live Storage API listing, not just trusting local caches), zero-byte-file check clean, only one filename anywhere has unusual characters (a pre-existing `&amp;`-in-URL artifact from monster's source data, unrelated to recovery, round-trips fine).
- Audited full bucket contents (not just pipeline images) before the user wiped storage: found `packages/`, `prescriptions/`, `rx/`, `tbyb/` folders in the bikershades bucket plus UUID-named folders in prosport/monster. `packages/` and `prescriptions/` were already backed up elsewhere (git, local files). `rx/`/`tbyb/`/UUID folders confirmed by the user to be their own test data, not real customer submissions — safe to lose.

## Full DB + storage reset and reimport

1. User ran `drop_schema.sql` then reapplied `001`–`005` (holding the `tbyb_packages` seed insert, since it requires the `bikershades` brand row to exist first — FK dependency, expected to fail before import).
2. User deleted all 3 Supabase Storage buckets.
3. Ran `upload_items.py` for all 3 brands — **first attempt silently failed**: buckets were freshly empty, but the local `upload_map.json` files (stale from the earlier recovery) made the script think everything was already uploaded, so it skipped every file (`0 uploaded, N skipped`) without erroring. Caught this by verifying a URL directly against the live bucket. Fixed by clearing the local `upload_map.json` caches to `{}` and re-running for real — verified after via live bucket listing cross-checked against `upload_map.json` and local files (exact match, zero discrepancies, all three brands) plus random live-URL spot checks.
4. Ran `import_items.py` **one brand at a time** per user's request. Prosport completed cleanly (150/150). Monster's import was intentionally killed mid-run to demonstrate crash recovery — left a dangling `brands` row with 0 products (the brand `INSERT` has its own standalone `conn.commit()` before the product loop starts, so it survives; the product loop only commits every 50 items in one transaction, so anything since the last checkpoint rolls back atomically on disconnect). Confirmed live against the DB, then re-ran monster cleanly (256/256) — the wipe step's `brand_slug`-scoped delete self-heals any partial state automatically.
5. Bikershades imported cleanly (464/464), with the same known 31-image gap surfacing as `WARN unresolved` messages during import (products still imported, just missing those specific image rows).
6. Ran the `tbyb_packages` seed insert after re-uploading the 6 package images manually — verified all 7 rows land correctly with live image URLs.
7. Fixed `prescription-frames/migrate.py` — it read from a nonexistent `.env.local` and expected `SUPABASE_SERVICE_ROLE_KEY`; pointed it at the real `.env` and `SUPABASE_SERVICE_KEY` instead. Ran it — 86/86 frames migrated, verified against DB row count and live image URLs.

## Final verification (all green)

- 3 brands, correctly named/slugged, no orphans.
- RLS enabled on all 18 public-schema tables; all 10 triggers present and attached.
- Per-brand completeness, cross-checked against `items.json` + the resolved image maps — **exact match, zero mismatches** on every table for all three brands:

  | Table | prosport | monster | bikershades |
  |---|---|---|---|
  | products | 150/150 | 256/256 | 464/464 |
  | categories | 18/18 | 19/19 | 23/23 |
  | product_categories links | 177/177 | 286/286 | 504/504 |
  | product_images | 507/507 | 822/822 | 1779/1779 |
  | description_images (distinct) | 2/2 | 10/10 | 31/31 |
  | product_description_images links | 134/134 | 250/250 | 722/722 |
  | variations | 1498/1498 | 2628/2628 | 5507/5507 |
  | variation_images | 24/24 | 125/125 | 1936/1936 |

- Every product has ≥1 `product_images` row and ≥1 category link (0 exceptions on all three brands).
- `tbyb_packages`: 7/7. `prescription_frames`: 86/86, live image URLs confirmed.
- Every non-seeded table (`admins`, `orders`, `cart_items`, `bookmarks`, `order_items`, `tbyb_submissions`, `rx_orders`) confirmed empty — no stray data from the reset process.

## Known permanent gaps (not fixable without source-site cooperation)

- **monster**: 1 image (`Gold-Blue-Green-Angle.jpg`, product `carmen-fashion-sunglasses`) — SSL cert on `sunglassmonster.com` doesn't match its own hostname.
- **bikershades**: 31 images across various products — `bikershades.com` returns `403` on these specific paths (mix of product angle shots and head-size sizing diagrams).
- Both are secondary images only — every affected product still has at least one working photo, per the "every item has ≥1 product image" coverage check that passed both before and after recovery.

## Backup

User copied `items.json`, downloaded images, `download_map.json`, and `upload_map.json` (post-reimport, so URLs are current) for all 3 brands, plus `prescription-frames.json`/images, into `sgm-data` (both on Desktop and inside the repo — the repo copy is currently untracked and **not gitignored**, worth adding to `.gitignore` if it shouldn't ever be committed).
