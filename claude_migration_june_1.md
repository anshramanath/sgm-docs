# What I Learned Building This Migration (June 1, 2026)
_Last updated: 2026-06-01_

A personal retrospective on what I learned and built migrating BikerShades from WooCommerce to Supabase — the decisions, surprises, and things I'd do differently.

---

## What I Built

A 7-step data migration pipeline in Python that takes a live WooCommerce store and seeds a clean, normalized Supabase/Postgres database. Every step is crash-safe, resumable, and has a matching QA script that audits and verifies the output before moving on.

The pipeline handles 647 products, 6,064 variations, ~2,500 images, and a full category tree — all sourced from a public WooCommerce Store API, transformed into a clean schema, and imported with zero broken references.

---

## WooCommerce APIs: Store vs Admin

The most practically important thing I learned was that WooCommerce has two entirely separate APIs:

**Store API** (`/wp-json/wc/store/`) — public, no auth, designed for the storefront cart UI. Returns a trimmed-down view of product data. Variation images are limited to 1 per variation (the featured image). This is what the public internet can access.

**Admin REST API** (`/wp-json/wc/v3/`) — requires auth (consumer key + secret generated in WooCommerce admin). Returns the full data layer — all images, all metadata, stock quantities, everything the merchant configured. The image shape is even different: `image` (singular featured) + `gallery_image_ids` (array of media IDs, not URLs).

The lesson: **don't assume a public API gives you complete data.** The Store API is lossy by design. We discovered this mid-project when a product showed 1 variation image in our data but 3+ on the live site. The only fix is the admin API, which requires credentials from the owner.

---

## Data Quality Is Never What You Expect

Raw WooCommerce data has a lot of quirks:

- **`on_sale` is unreliable** — WooCommerce sets it true even when `sale_price == regular_price`. We ignore it entirely and compute sale status from the price comparison ourselves.
- **Power slugs were inconsistent** — the same diopter value was stored as both `1-00` and `1.00` across different products. Had to normalize to decimal format.
- **Product attribute terms had unused entries** — WooCommerce lets you add color terms to a product without actually creating a variation for that color. Stripping unused terms during reshape prevents ghost color swatches on the frontend.
- **Variable product SKUs are meaningless** — WooCommerce lets you set a SKU on the parent product of a variable product, but it's never used. The real SKUs are on the variations. Keeping the parent SKU caused collisions at import time.
- **HTML everywhere** — names, descriptions, short descriptions all arrive with HTML entities and tags. Can't store raw HTML in a clean DB.

---

## The Value of Bucketing

Rather than trying to fix every problem in place, I built a bucketing system: products that fail any check get routed to a named problem bucket instead of crashing the script or silently dropping the product. Every product across all 8 buckets always sums to 647 — nothing is ever lost.

This was the right call. It meant:
- The pipeline could run end-to-end even with imperfect data
- Problems were categorized, not just logged
- The owner gets a clean list of exactly what needs their attention and why
- QA scripts could verify no products went missing between steps

The order of bucket checks matters too — putting zero-price first (before any reshape validation) avoids false positives on internal/service products that might have other issues.

---

## Global SKU Uniqueness Is Hard to Catch Early

The DB had a `UNIQUE` constraint on `variations.sku`. I didn't enforce this during reshape, so the first import run crashed mid-way with a `UniqueViolation` error. The culprit was a shipping label product where the parent SKU was identical to all 5 variation SKUs.

The fix was a post-processing pass after reshape: walk every product in every bucket, pool all SKUs (product + all variation SKUs) into a global set, move any product with a collision to a `duplicate_sku_items` bucket. The post-processing approach was smarter than interleaving the check into the main loop because zero-price products use `continue` in the loop and would have been skipped.

Lesson: **enforce DB constraints at the data transformation step, not at import time.**

---

## Crash-Safe Design

The download and upload steps can take a long time and hit network failures. Both scripts use JSON checkpoint files (`url_map.json`, `upload_map.json`) that are updated after every single file. On restart, the script reads the checkpoint and skips already-completed work.

This meant a script killed mid-run could be restarted and would pick up exactly where it left off, with zero duplicate work and zero missing files. The QA scripts then verify the checkpoint maps are complete and consistent.

---

## Image Deduplication

WooCommerce stores the same image file at the same URL and references it from multiple products and variations. If you naively download by product, you download the same file many times.

The solution was deduplication by URL — `url_map.json` is keyed by the full source URL. Before downloading, check if the URL is already in the map. If yes, skip it. Same for upload. This means a single physical image file, no matter how many products reference it, is downloaded once, uploaded once, and resolved to one Supabase URL used by all referencing products.

---

## The QA Pipeline

Every step in the migration has two scripts: an **audit** (does the output look internally valid?) and a **verify** (does the output match the input without data loss?). Running both after each step before moving on caught several bugs:

- Verify after reshape caught that products were going missing between runs
- Verify after import caught that variation image expected counts needed to exclude permanent 404s
- Audit after recategorize confirmed all paths terminated at leaf nodes

The pattern of audit + verify is more valuable than a single combined check because they catch different failure modes: audit catches structural corruption, verify catches silent data loss.

---

## Schema Decisions That Paid Off

**`stock int` instead of `in_stock boolean`** — initialized from `is_in_stock` as 0/1, but the owner can update it to real quantities later without a schema migration. Frontend just checks `stock > 0`.

**`variation_images` table** — instead of a single `image_src` column on variations, a separate table with `sort_order` allows multiple images per variation. The Store API only gave us 1, but the schema is ready for when we switch to the admin API.

**Nullable price and unit fields** — rather than bucketing every product with missing weight or bad dimensions into a problem bucket, null them and let the owner fill them in via the admin dashboard. Reduced the problem bucket count significantly without hiding the issue.

**Category tree from URLs** — WooCommerce category links encode the full hierarchy in the URL path. Storing the link URL in reshape and parsing it during recategorization meant the reshape step didn't need to know anything about the category tree structure — that concern was isolated to one script.

---

## What I'd Do Differently

**Use the admin REST API from the start.** The Store API was convenient (no credentials needed) but the missing variation images are a real gap. Getting credentials from the owner upfront would have avoided a re-fetch step.

**Enforce all DB constraints in reshape, not import.** The SKU uniqueness bug taught me this. Before writing the import script, I should have read every constraint in the schema and added a reshape check for each one.

**Version the checkpoint files.** `url_map.json` and `upload_map.json` are overwritten each run. If a run produces a bad map, there's no rollback. A simple versioned copy (or git-tracked intermediate) would make debugging easier.
