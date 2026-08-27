# Staging Migration Summary (August 24, 2026)

Full migration of the catalog, TBYB packages, and prescription frames to a newly created staging Supabase project — a mirror of the production migration done earlier, run against a second, independent project.

## How environment switching works in this repo

There's no built-in staging/prod split — the pipeline reads a single global `.env`, and "which environment" is entirely determined by whatever `SUPABASE_URL`, `SUPABASE_SERVICE_KEY`, and `OFFICE_DATABASE_URL` currently hold. Pointing the whole pipeline at staging is just a matter of overwriting those three values with the staging project's own. No code changes needed — this is intentional simplicity, but it means there's zero safety net distinguishing the two: whichever credentials are in `.env` at the moment a script runs is where it goes, full read/write/DDL access, no confirmation prompt.

## What was built

1. **Schema applied to staging** — `001`–`005` run against the new project (via Supabase's dashboard SQL editor), producing an identical 18-table schema to prod, RLS enabled everywhere, all 10 triggers attached.
2. **Images uploaded to staging's own buckets** — `upload_items.py` run for all three brands (355/705/2224 files), reusing the *same* locally-downloaded images from the prod migration (no need to re-fetch from source, since the images themselves aren't environment-specific — only where they get uploaded to is).
3. **Catalog imported to staging** — `import_items.py` run for all three brands in one pass (unlike prod, which was done one-at-a-time to demonstrate crash recovery). All three completed cleanly first try.
4. **TBYB packages seeded and image URLs corrected** — package images uploaded, seed insert run, then a follow-up fix (see below) to repoint `image_src` at staging's own bucket.
5. **Prescription frames migrated** — `migrate.py` run against staging, 86/86 frames.
6. **Full end-to-end verification** — every table's row count cross-checked against `items.json` for all three brands, TBYB/prescription-frames `image_src` values confirmed to point at staging (not prod), non-seeded tables confirmed empty.

## Bugs caught along the way

**1. Direct connection string doesn't resolve.** The first connection string pulled from Supabase's dashboard was the direct-connection format (`db.<project-ref>.supabase.co`), which is IPv6-only and failed to resolve on this network — a gotcha already documented in this repo's own `PIPELINE.md` for the *production* connection, which turned out to apply identically to any new project. Fix: use the session pooler connection string instead (`aws-*.pooler.supabase.com`, username `postgres.<project-ref>`).

**2. Partial `.env` swap caused a false alarm about touching prod.** `SUPABASE_URL` got updated to staging first, but `OFFICE_DATABASE_URL`/`HOME_DATABASE_URL` (the direct Postgres connection strings) still referenced prod's project ref. A verification query run through `OFFICE_DATABASE_URL` at that point landed on prod (read-only — just table/row counts), and its results were briefly misattributed as evidence that a schema-reapply action had hit prod. In reality that action had gone through Supabase's dashboard SQL editor — a completely separate access path from anything in `.env` — and had correctly landed on staging the whole time. The lesson: when multiple credential fields exist for the same "environment" (a REST API URL/key pair *and* a separate direct DB connection string), all of them need updating together, and a dashboard action is never observable through an unrelated env-var-based connection — don't conflate the two when reasoning about what a given check actually proves.

**3. Hardcoded prod URLs leaked into staging via seed data.** `004_tbyb.sql`'s TBYB package seed `INSERT` hardcodes full image URLs including the production project ref. Run unmodified against staging, the 7 rows landed with `image_src` pointing at prod's bucket — which "worked" visually (prod's images are public and still live) but meant staging was silently depending on production infrastructure rather than being self-contained. Fixed with a straight `UPDATE ... SET image_src = replace(image_src, '<prod-ref>', '<staging-ref>')`, since the same files already existed at the same paths in staging's own bucket. `prescription-frames/migrate.py` didn't have this problem — it builds `image_src` dynamically from `env['SUPABASE_URL']` rather than hardcoding it, so it was correct on staging without any fix needed. Worth applying that same pattern to the TBYB seed data if it's ever reused for a third environment.

**4. Stale local caches strike again.** Same bug as during the prod recovery — `upload_map.json` still had prod's public URLs cached locally, which would have made `upload_items.py` think everything was already uploaded to staging's (empty) buckets and skip every file. Caught and cleared *before* running this time, rather than after discovering it failed silently.

## Final verification (all green, exact match with prod)

| Table | prosport | monster | bikershades |
|---|---|---|---|
| products | 150/150 | 256/256 | 464/464 |
| categories | 18/18 | 19/19 | 23/23 |
| category_links | 177/177 | 286/286 | 504/504 |
| variations | 1498/1498 | 2628/2628 | 5507/5507 |
| product_images | 507/507 | 822/822 | 1779/1779 |
| variation_images | 24/24 | 125/125 | 1936/1936 |
| description_images | 2/2 | 10/10 | 31/31 |
| description_links | 134/134 | 250/250 | 722/722 |

Plus: `tbyb_packages` 7/7 (all pointing at staging's own bucket), `prescription_frames` 86/86 (all pointing at staging's own bucket), RLS on all tables, all 10 triggers present, every non-seeded table empty.

## Reusable lesson for next time

Before running anything against a specific environment — especially the first write of a session — verify the *actual* project ref embedded in the connection string/URL currently in `.env`, not just "did I paste something in." A partial credential swap (one var updated, another forgotten) is easy to do and easy to miss, since the pipeline has no built-in guardrail against it.
