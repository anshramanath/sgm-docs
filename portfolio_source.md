# Sunglass Monster Platform Rebuild — Portfolio Source Material

*Compiled from 154 session write-ups (ChatGPT + Claude conversation logs) and ~85 screenshots saved between May 27 and August 24, 2026. This is raw material for a portfolio page, not a finished draft — organized and de-duplicated so it can be handed to a design pass (e.g. Claude Design) without re-reading the original 130+ files.*

*Format note: this is deliberately **not** written as a polished agency-style "case study." The intended shape is more personal — "what I did, what I learned" — since the material is stronger read as genuine reflection than as marketing copy for a solo student project. The **Six Did/Learned Threads** section below is the recommended backbone; everything else is supporting detail underneath it.*

---

## How to use this document

1. **Project Overview** and **By the Numbers** are the pitch-ready summary — background/stats for wherever the page opens.
2. **Six Did/Learned Threads** is the recommended organizing structure — six self-contained "what I did → what happened" pairs spanning the whole project, each with suggested screenshots. Build the page around these rather than a chronological case-study narrative. Note: thread #3 (frontend/state management) was absent from an earlier draft of this document and was added back in after a review pass flagged that the project's frontend/backend work — Suspense, debounce, the provider stack — wasn't represented anywhere.
3. **Timeline** underneath is the full chronological build story in nine phases, condensed from day-by-day session notes — kept as backup detail for filling out any thread above, not meant to structure the page itself.
4. **Photo Manifest** at the end lists every screenshot, grouped by what it shows, so they can be matched to the right thread.
5. A note on ordering: file "date modified" timestamps on disk turned out to all be from the same ~3-hour window (the files were copied into one folder right before this was compiled), so they weren't usable for ordering. Every date below instead comes from the date written inside each file's own text.
6. **Before publishing:** this was built for a paying client (the Sunglass Monster owner), not a personal side project — real production domains, real Stripe account, live customer data. Confirm it's okay to publish real screenshots, domain names, and specifics (especially admin/analytics screenshots that could show order or revenue data) before the page goes live; anonymize brand names if there's any doubt.

---

## Project Overview

Over one summer (late May through late August 2026), a full e-commerce platform was designed and built from scratch to replace three aging WooCommerce storefronts — **BikerShades**, **ProSport Sunglasses**, and **Sunglass Monster** — with a single modern, shared platform (internally named **Sunglass Monster** as the platform, with BikerShades launching first).

The work spanned the full stack of a real production system:

- A **17-step, crash-safe data migration pipeline** (Python) that pulled ~1,650 raw products and thousands of variations out of three legacy WooCommerce catalogs, validated and cleaned years of accumulated data debt (duplicate SKUs, mixed product families, private/draft junk listings), and landed a clean, verified catalog of **870+ products** in a new Postgres (Supabase) database.
- A shared **Next.js 16 backend/API** serving three storefronts, an authenticated user API (cart, bookmarks, orders, checkout), and an admin API — all off one multi-brand-aware schema.
- Three **Next.js storefronts** (one per brand) sharing one codebase, with full product browsing, variant selection, auth, cart, bookmarks, and checkout.
- A custom **admin dashboard** (orders, products, categories, analytics, plus feature-specific admin views) designed with its own editorial black-and-white visual system before a line of component code was written.
- **Stripe Checkout** integration with webhook-driven, idempotent order creation, atomic sales counters, and a full refund lifecycle.
- Two BikerShades-specific commerce features built end-to-end: **Try Before You Buy** (pay a refundable deposit, try prescription frames at home) and **Rx Frame Ordering** (a full prescription-frame purchase flow with deposit credit).
- Integrations with **Veeqo** (inventory/fulfillment sync) and an evaluation of **Sellbrite**, plus **Resend** for transactional email.
- A full **production launch**: custom domain cutover from WordPress/SiteGround to Vercel, live Stripe mode, a maintenance-mode kill switch, and a from-scratch, zero-mismatch production data reimport — followed by a staging environment mirroring production.

The project's own internal documents describe its central discovery well: it started as "migrate WooCommerce" and became "determine the authoritative source of truth for every piece of product data" — the legacy system, the inventory system, and the new database each turned out to be right about different things, and figuring out which was which shaped almost every technical decision that followed.

---

## By the Numbers

- **~13 weeks** of documented work, May 27 – August 24, 2026
- **154** session write-ups + **~85** screenshots saved along the way
- **3** storefront brands rebuilt on one shared codebase (BikerShades, ProSport, Sunglass Monster)
- **870+** products migrated cleanly into the new database (from ~1,650 raw WooCommerce listings across all three brands)
- **86** prescription frames migrated for the Rx ordering flow
- **9,600+** product variations, **3,000+** product images imported
- **Zero mismatches** in the final production data verification across every table, every brand
- Two full custom commerce flows shipped beyond a standard storefront: **Try Before You Buy** and **Rx Frame Ordering**
- Full production launch with live payments, custom domains, and a mirrored staging environment

---

## Tech Stack

- **Frontend:** Next.js 16 (App Router), React Server Components, Tailwind CSS, shadcn/ui (components owned in-repo, not black-boxed)
- **Backend:** Next.js API routes (three roles: public API, authenticated user API, admin API)
- **Database:** Supabase (Postgres + Storage + Auth), Row Level Security throughout
- **Payments:** Stripe Checkout, webhook-driven order creation
- **Inventory/Fulfillment:** Veeqo (evaluated Sellbrite as an alternative)
- **Email:** Resend (custom SMTP for transactional auth email)
- **Hosting:** Vercel
- **Migration tooling:** Python pipeline (fetch → validate → reshape → recategorize → download → upload → import), plus a custom MCP server driving Claude Desktop for AI-assisted product content generation

---

## Six Did/Learned Threads

*Six threads pulled directly from the timeline below, one per major area of the project. "Did" is a compressed restatement of what the session write-ups documented. "From the notes" contains only decisions, rationale, or observations that the source material itself recorded at the time — nothing here is added interpretation; it's extraction and compression of what's already in the Timeline/batch write-ups. Rewrite the wording for the page, but the facts and quotes below should trace back to a specific dated entry.*

### 1. Data migration and the SKU forensics problem

**Did:** Built a pipeline that pulled ~1,650 products off three legacy WooCommerce stores and audited every field against multiple sources (WooCommerce, Veeqo, the new DB) to determine which one was authoritative for each piece of data — including a single-letter SKU prefix (`C` for combo products) whose mishandling was worth 12 products and 437 variations once fixed (June 4).

**From the notes:** The source material states this discovery twice, in almost the same words each time. The project overview describes it as: "it started as 'migrate WooCommerce' and became 'determine the authoritative source of truth for every piece of product data.'" The June 6 session notes restate it independently: "WooCommerce's only truly trustworthy artifact turned out to be SKUs, not the product data around them — the project shifted from 'migrate WooCommerce' to 'determine which SKUs represent real sellable products.'"

**Screenshots:** `database-001-004-overview.png`, `database-cart-bookmarks-tables.png`, `prod-admin-products-page.png`

### 2. Payments — Stripe webhooks and idempotency

**Did:** Built order creation entirely off Stripe webhooks (never at checkout-session creation), with two independent layers of idempotency — a Stripe-side idempotency key for double-clicks, a DB-side unique constraint for duplicate webhook deliveries — plus atomic Postgres RPCs for sales counters so concurrent writes can't lose an update (June 18).

**From the notes:** The June 18 write-up records the reasoning for each piece directly: orders are created only in the webhook specifically so a duplicate Stripe event can't create a duplicate order; the RLS design makes orders/order_items select-only for authenticated users, with the webhook (service-role key) as the only write path, "making forged orders impossible via direct Supabase calls." It also documents a deliberate simplification made the same day: no stock gating at checkout — orders go through regardless, and the owner refunds manually if something can't be fulfilled.

**Screenshots:** `prod-checkout-page.png`, `prod-stripe-checkout.png`, `stripe-payment-setup-call.png`

### 3. Frontend state — providers, streaming, and the auth-flash bug

**Did:** Built cart/bookmark/auth state as a nested provider stack (`AuthProvider` → `CartProvider` → `BookmarkProvider`) with `Map`-based state, 800ms-debounced DB writes, and Suspense-streamed category pages; separately diagnosed and fixed a bug where a background 401 could leave the header showing stale auth state after a session had actually expired (June 26, August 4).

**From the notes:** The August 4 write-up documents this as "The Auth Flash Problem": logged-in users briefly saw "Sign In" on every page load before it flipped to their profile icon, because the navbar combined a server-accurate `isSignedIn` boolean with a client-side `loggedIn` state that always started `false` and only became accurate after a `useEffect` ran. The recorded fix: change `loggedIn`'s initial value from `false` to `null` ("not yet determined"), then combine the two as `isSignedIn && (loggedIn ?? true)`. The notes describe it as "a one-line, low-risk change that correctly models a genuine three-state system (undetermined / true / false) that had been wrongly collapsed into two," requiring no changes anywhere else since `null` is falsy.

**Screenshots:** `frontend-bookmarks-local-storage.png`, `backend-nextjs-debugger.png`, `claude-design-frontend-products.png`

### 4. Try Before You Buy — kept outside the main cart

**Did:** Designed and shipped a full feature — deposit packages, prescription intake, Stripe checkout, admin fulfillment — built as its own flow rather than routed through the existing cart/checkout code (July 29–31, August 1–7).

**From the notes:** The July 29 design notes state the architectural call directly: "TBYB deliberately does not go through the existing cart, since adding nullable branching throughout cart code for one single-brand feature wasn't worth the complexity." The July 31 notes record a related, separately-made decision: the submissions table stores a full snapshot of the package at submission time with no foreign key back to the live catalog, "so historical submissions must stay stable even if a package is later renamed, repriced, or deleted."

**Screenshots:** `prod-tbyb-page.png`, `prod-tbyb-form.png`, `claude-design-admin-tbyb.png`

### 5. Product copy generation via an MCP server

**Did:** Built a FastMCP server that Claude Desktop drives over stdio to generate descriptions for 882 products across 3 brands from their actual product photos, with a `save_content` validator that rejects entries over 80 words, containing raw measurements, or using banned marketing phrases (June 7).

**From the notes:** The June 7 write-up records why the validator exists: the rules were iterated on "after catching the model inventing detail (e.g. calling plain-lens computer readers 'blue-light blocking' just because the image looked that way)" — i.e., the prompt instructions alone weren't sufficient, which is why a code-level check was added on top of them.

**Screenshots:** `migration-mcp-prompt.png`, `migration-mcp-server.png`, `prod-product-page.png`

### 6. Launch — domain cutover and the production data reset

**Did:** Cut three live domains over from WordPress/SiteGround to Vercel, flipped Stripe to live mode, and ran a full reset and reimport of the production database and Storage — during which all three original legacy WooCommerce sites became unreachable, with zero fallback available (August 22–23).

**From the notes:** The August 23 write-up documents the recovery mechanism directly: image filenames are "a deterministic pure function of the source URL," and Supabase Storage "never deletes or re-encodes anything" — so every image was re-derived and re-downloaded byte-for-bit-identical from Storage instead of the dead source sites. Recorded result: "100% of what existed before" recovered (355/355, 705/706, 2224/2255 — the two shortfalls noted as pre-existing gaps, not new losses), and the final production verification shows "zero mismatches across every table for all three brands."

**Screenshots:** `frontend-prod-503-maintainence-redirect.png`, `prod-sunglass-monster-landing-page.png`, `database-prod-005.png`

---

## Timeline

### Phase 1 — Migration Pipeline Kickoff (May 27–31, 2026)

## May 27–31, 2026

### May 27 — Discovery and architecture scoping

**Track:** Migration pipeline (planning)

- Investigated the live bikershades.com storefront and discovered it ran on WordPress + WooCommerce, with a public, undocumented Store API at `/wp-json/wc/store/products`. This reframed the project from "scrape a website" to "migrate a data source" — a much better starting position.
- Mapped out the intended pipeline: WooCommerce API → `products.json` (raw backup) → download images locally → reshape into a clean schema → rehost images → insert into own DB → build new storefront. Decided early to always download and rehost images rather than hotlink the old WordPress URLs, to avoid dependency on the legacy site staying alive.
- Went deep on Supabase fundamentals in prep for the new backend: JWTs, `getUser()` vs. session state, Row Level Security (`USING` vs `WITH CHECK`), and when to use the `service_role` key (migrations, webhooks, admin tools, bulk imports) vs. `anon` + JWT (user-owned CRUD).
- Explored a tangent: bikershades also sells on Amazon, raising a future possibility of an MCP server wrapping the Amazon SP-API for AI-driven inventory management. Parked as a later phase, not started.
- Sketched an 8-phase rebuild plan (fetch → images → normalize → schema → upload → import → storefront → possible Amazon integration) and produced a migration guide artifact as the reference doc for the work to come.

### May 28 — Pipeline scripts running for real; multi-brand architecture decided

**Track:** Migration pipeline + system architecture

- Wrote and ran `fetch_products.py` against the WooCommerce Store API — pulled all **647 products across 7 pages**. Learned that `per_page` is a server-protection pagination limit, unrelated to what the storefront displays, and that parent-product pagination deliberately excludes variations.
- Wrote `fetch_variations.py` to pull full variation data (price, stock, images, SKU) per variation ID, since the main endpoint only returns stub attribute values. Built it crash-safe from the start — checkpoints after every variation, resumes by diffing what's already saved. Result: **6,064 variations across 486 variable products**.
- Wrote `reshape_products.py` and ran a first pass: 639 clean products (8 filtered for zero price). Nailed down several data-shape gotchas that would recur all week: prices are strings in cents, `on_sale` alone doesn't mean actively discounted (must also check `sale_price < regular_price`), `stock_status` is always null (real signal is `is_in_stock`), and 9 simple products had a WooCommerce misconfiguration flag (`has_variations: true`) that had to be ignored.
- Big architecture decision session: originally considered one frontend per brand each with its own backend/MCP server, but rejected that as duplicative. Settled on **3 separate Next.js frontends sharing 1 backend, 1 MCP server, and 1 multi-brand-aware Supabase database** (`brand_id` on every relevant table from day one). Also decided the frontends would use server-side rendering rather than pure client-side fetching, weighing better SEO against the fact that GitHub Pages (which can't run SSR) was ruled out as a host in favor of a real Node server.
- Plan: build and validate BikerShades first, then reuse the same backend/DB/MCP/admin dashboard for the other two brands.

### May 29 — Backend scaffolded, schema finalized, first successful DB import

**Track:** Migration pipeline + backend/admin dashboard build

- Backend build kicked off for real: a Next.js 16 app serving three roles — admin dashboard, API server (frontends never touch Supabase directly), and (planned) MCP server. Built out the route structure, two-Supabase-client pattern (`service_role` for all data ops, session client for reading the logged-in user's cookie), and a shared `requireAdmin()` guard on every write route (auth success alone isn't enough — the user also needs a row in the `admins` table).
- Hit and fixed several real infrastructure gotchas: `cp -r` silently broke `node_modules/.bin` symlinks when the project was copied from a temp directory (fixed by wiping and reinstalling); Next.js 16 renamed `middleware.ts` → `proxy.ts` with a hard error if the exported function is still named `middleware`; Supabase's `.single()` returns error code `PGRST116` (not a generic 500) for "no row found," so routes had to special-case it into a 404; Next.js 16 made `searchParams`/`params` async (`Promise`), requiring `await`.
- Finalized the DB schema (9 tables: brands, categories, products, product_categories, variations, product_images, variation_images, description_images, admins), with `attributes` stored as `jsonb` grouped by type so listing pages can filter without joining variations.
- Ran the pipeline further: uploaded images to Supabase Storage (dedup via `upload_map.json`, one timeout that self-healed on retry) and successfully **imported 589 products, 63 categories, and 1 brand into Supabase** — the first real data landing in the new database, exit code 0.
- Hit and fixed a real process bug: the first image-download run used product **slug** for folder names instead of SKU (the DB's actual unique key); had to stop the job, wipe the `images/` folder, and restart cleanly rather than try to migrate the mismatched map.
- 58 products flagged for manual owner review across 6 issue buckets (invalid SKU, no weight, zero price, missing attributes, duplicate names, no images) — first pass at what would become a recurring "owner escalation" list.

### May 30 — Pipeline hardening, network lessons, first true end-to-end run

**Track:** Migration pipeline (QA, reliability) + backend handoff docs

- Ran an audit pass over the clean dataset and caught a real bug: two products had `weight = 0.0`, which passed a `weight is None` check but was still functionally missing. Tightened the check to `not weight` and moved them to the flagged bucket, dropping clean count from 589 → 586.
- Removed the separate `variation_images` table after auditing showed 5,756/5,784 variations have exactly 1 image (the handful with more were a WooCommerce misconfiguration, not real data). Flattened to `image_src`/`image_name` columns directly on the `variations` row, with a new `multiple_variation_images` bucket to catch the outliers during reshape.
- Interesting technical detour: profiled the fetch scripts and learned **network latency, not the polite `time.sleep()` calls, was the real bottleneck** (200–500ms per request regardless). Tried `ThreadPoolExecutor` concurrency (10, then 3 workers) to speed up variation fetching — bikershades.com immediately started returning mass 503s. Root cause: the server rate-limits by *concurrent connections*, not request rate, so sequential was actually correct. Reverted.
- Found and fixed a subtler reliability bug: killing a script mid-`json.dump()` could leave a corrupted checkpoint file. Fixed with the atomic-write pattern (write to `.tmp`, then `os.replace()` — an indivisible OS rename) across all checkpointing scripts, plus exponential-backoff retry on 503s.
- Reworked the pipeline into a formal 7-step structure with a new **recategorize** step (step 4) between reshape and download — WooCommerce's category data was messy (structural "see all" nav nodes, single-item leaves, non-leaf product assignments), so a rules engine (drop → merge → collapse single-leaf iteratively → strip non-leaf paths) rebuilt a clean leaf-only category tree. Also redesigned image filenames to use the full URL path after `/uploads/`/`/gallery/` (not just the trailing filename) after discovering collisions from WordPress's date-based upload folders.
- Built out a proper QA layer: paired `audit_*` (internal validity) and `verify_*` (cross-reference against source) scripts for every pipeline stage, with docs written as human-readable summaries rather than raw output dumps.
- Milestone: ran the full pipeline end-to-end for the first time on the recategorized dataset — **581 products, 5,649 variations, 54 categories imported, all QA checks passing.** Reorganized the repo into `pipeline/`, `qa/`, and `db/` folders and added a `CLAUDE.md` for session continuity.

### May 31 — DB fully seeded, frontend hits a template milestone, backend API finalized, admin UI prototype

**Track:** Admin dashboard build + frontend storefront + backend public API

- Wrote a comprehensive admin dashboard handoff against the now-stable, fully seeded database: 581 products, 5,649 variations, 54 categories (max depth 3), 2,618 product images, 935 description images. Documented real data-quality quirks that would shape the admin UI: some attribute slugs aren't URL-safe (`"Matte Black"` stored with a space and mixed case), and color terms exist as both `black` and `Black` — flagged as something to ask the owner about rather than silently merge.
- Interesting build detour on the admin dashboard: instead of a conventional table-based UI, built the category browser as an **interactive node graph** using React Flow (`@xyflow/react`) with `dagre` for automatic tree layout — each top-level category tree laid out independently and placed side by side, with clickable "product count" nodes that slide up an inline product panel (no modal/overlay). Styled with a distinct "Profiler" design system (Source Serif 4 + Courier Prime, inline styles over CSS custom properties, no Tailwind despite it being installed). Also had to move the `admins` table check out of the Next.js proxy/middleware entirely, since it needs the `service_role` key which can't run in the Edge Runtime — middleware there does session-presence checks only.
- Finalized the public backend API: revealed the project's actual multi-brand platform name, **Sunglass Monster**, with BikerShades as the first brand launching on it. Locked in an architecture where the frontend never queries Supabase directly — only POST endpoints with JSON bodies, camelCase responses over a snake_case DB, and IDs (not slugs) as the source of truth for both categories and products, since product slugs turned out not to be unique. RLS was re-enabled after being disabled earlier in the week. Tested endpoints for brands, category tree, product search (573 products / 115 pages, pagination and filtering all correct), and product detail all passed.
- Frontend storefront reached a genuine milestone: category tree, search, product listings, product detail with variation selection, image gallery with fallback, breadcrumbs, and homepage were all built out, at which point the team flagged **"STORE FRONT TEMPLATE CREATED HERE"** — the intended fork point for building the other two brands' storefronts without touching core architecture again.
- Fixed several real frontend/backend contract bugs along the way: variation attributes were coming through as raw slugs (`{name: "black-mirror-green", value: "black-mirror-green"}`) instead of human-readable pairs, breaking variation selection — fixed by having the backend map slugs through the product's attribute definitions. A "Specifications" section was accidentally rendering the internal attribute-definition array as if it were product specs; fixed short-term by hiding it, then removed `product.attributes` from the public API entirely. Also moved category-tree fetching from a client-side `useEffect` into the root server layout after browser fetches to it were failing.
- 66 items remained parked for owner review across 8 flagged buckets (invalid SKU, no weight, zero price, missing attributes, duplicate names, no image, no category, single-leaf) — the recurring theme of the week: the pipeline is solid and automated, but a chunk of the catalog needs a human decision before it can go live.

### Phase 2 — Multi-Brand Migration + Architecture Decisions (June 1–7, 2026)

## June 1–7, 2026

### June 1 — ProSport site recovery + BikerShades pipeline retrospective

Two very different tracks landed on the same day.

**Ops/recovery (ProSportSunglasses):** the legacy WordPress/WooCommerce install was throwing "critical error" on `wp-admin` and the public site was unstable. Root-caused to two independent issues: (1) a corrupted/missing plugin was crashing WP on boot — found by renaming plugin folders until a staging copy loaded, then re-activating only the essentials (WooCommerce, Elementor, Astra) and leaving Sellbrite/Veeqo/Stock Sync/shipping plugins deactivated; (2) once the site booted, the public storefront still rendered broken (missing hero, collapsed layout) while the logged-in admin view looked fine — traced to stale Elementor-generated CSS/cache, fixed via Elementor "Clear Files & Data" + "Sync Library" + a SiteGround dynamic-cache purge. Confirmed no data loss: the WooCommerce DB, products, categories, and REST API keys were all intact — the outage was purely operational. This unblocked using the WooCommerce **admin** REST API (vs. the public Store API) for the ProSport migration.

**Migration retrospective (BikerShades):** a personal write-up on the first-generation 7-step WooCommerce→Supabase pipeline, covering 647 BikerShades products, 6,064 variations, ~2,500 images. Key lessons banked here that shaped everything after: the WooCommerce **Store API is lossy** (max 1 image per variation, binary stock) vs. the **admin API** (full data, requires owner credentials) — this realization is what drove the API switch used for the rest of the summer; `on_sale` is unreliable and better computed from price comparison; a "bucketing" system routes failed products to named problem buckets instead of dropping them (all 647 always accounted for); SKU uniqueness has to be enforced at transform time, not import time, after a `UniqueViolation` crash mid-import; crash-safe checkpoint files (`url_map.json`, `upload_map.json`) make image download/upload resumable; image dedup by URL avoids re-downloading the same file referenced by multiple products.

### June 2 — Second-generation pipeline: ProSport run, folder rename, admin API adopted

Full day rebuilding the pipeline for ProSport using the newly-available admin REST API (credentials obtained after the June 1 recovery). Renamed the pipeline's folders/scripts for clarity and scoped all output to `data/{brand_slug}/` so brands never collide on disk.

**ProSport run-1 numbers:** 171 fetched → 169 validated products → 1,758 variations fetched → 1,740 validated → 166 items created → 12 categories, 197 leaf assignments → 411 images downloaded/uploaded (0 failed) → 166 products / 1,713 variations / 12 categories imported to Supabase. All verify scripts passed.

Key decisions locked in: prices always stored as integer cents; `dimensions` is always a full object even when null; variable-product min/max price derived from variation *regular* price (not sale price); attribute names get `"Choose "` prefix and `" for Filter"` suffix stripped; WooCommerce bifocal "Power" values normalized (`1-5` → `1.50`); variation images deduped against the parent gallery — 97.9% of ProSport variations ended up with no unique image of their own.

### June 3 — Multi-brand config, Monster run starts, schema decisions

Formalized the multi-brand architecture: a single `config/__init__.py` module with `BRAND_CONFIG` dict and `load_brand()`, selected via one `BRAND` env var — adding a brand becomes one dict entry plus `.env` credentials instead of touching 28 files.

Schema commitments made here: categories as a self-referencing table (`categories.parent_id`) built lazily via `ensure_category(path)`; variation attributes stored as raw `jsonb`; `total_sales` treated as "unknown" not "confirmed zero" when 0, since WooCommerce's counter resets across migrations; JSON field names in reshape output deliberately match DB column names so verify scripts can diff by name with no mapping table.

**ProSport finalized:** 166 products / 1,713 variations / 12 categories / 565 product images in Supabase. **Sunglass Monster run started:** 277 fetched → 267 validated → 265 items created.

### June 4 — BikerShades onboarded; SKU pattern deep-dive; Veeqo evaluated

The biggest single-day push of the batch — BikerShades (704 fetched products) entered the pipeline and immediately exposed much messier data than the other two brands.

**SKU forensics.** BikerShades variation SKUs follow a `{modelBase}-{frameColor}-{lensType}-{strength?}-{suffix?}` pattern; parent SKUs carry WooCommerce dedupe junk (`-0`, `-0-XT`, `-Master-MFB`, `-COMBOS`) that must never be used to derive the "real" product SKU — an early attempt caused 200+ false duplicate-SKU flags. Discovered that WooCommerce groups genuinely different products under one listing — polarized (`CAPL`) vs non-polarized (`CA`) variants, or old vs revised SKUs — and built a **"base SKU mismatch"** validation check. This immediately over-flagged legitimate **combo products** (`CPR29X36ST` vs `PR29X36ST`), so the check was patched with a "single leading-C" normalization rule — recovering **+12 BikerShades products / +437 variations** in one fix.

**Veeqo evaluated as a data source.** Coverage: BikerShades ~99%, Monster ~98%, ProSport only ~10%. Found Veeqo useful for exactly one thing — confirming a SKU is in active inventory — and unreliable for everything else (`qty_on_hand` populated on only ~8% of rows, 431 duplicate SKU rows with 162 genuine price conflicts). Decision: Veeqo becomes an informational/gating SKU check, never a data-enrichment source.

**Junk-SKU filtering added** for BikerShades specifically, catching shipping labels, Try Before You Buy deposits, and Rx frame listings implemented as individual WooCommerce products — flagged for a rebuild as first-class app features rather than migrated as catalog items. This is the first mention of what would become the TBYB and Rx Frames features built later in the summer.

**End-of-day create-items counts:** ProSport 161 items; Monster 260 items; BikerShades 461 items.

### June 5 — Veeqo gating added; SKU quality audit; family-SKU concept proposed

Formalized Veeqo as a **gating** pipeline stage: items whose SKUs are entirely absent from Veeqo get flagged and excluded. Final funnel after gating: ProSport 154 items, Monster 255 items, BikerShades 458 items — flagged items were overwhelmingly bifocals/readers whose high-diopter variants apparently were never synced to marketplace inventory.

**Strategic pivot proposed:** rather than treating WooCommerce as the long-term source of truth, the target architecture becomes `Amazon/eBay → Veeqo → Supabase → Next.js storefronts`. This is also where the **family SKU** concept was proposed: every variation SKU should decompose into a globally-unique `family_sku` + variation chunks, making product/variation grouping deterministic.

### June 6 — Family SKU / grouping problem restated; MCP exploration

A master-learnings recap reframes the summer's core insight: WooCommerce's only truly trustworthy artifact turned out to be **SKUs**, not the product data around them — the project shifted from "migrate WooCommerce" to "determine which SKUs represent real sellable products." Separately, an exploratory research session worked through MCP server fundamentals — a direct precursor to the content-generation MCP server built the very next day.

### June 7 — Pipeline rewrite (`final-migration`) + AI content-generation MCP server; frontend architecture set

**Pipeline rewrite** under a new `final-migration/` directory with a leaner stage list, new attribute-name consolidation across brands, and ~7,525 duplicate variation images removed globally.

**AI content-generation system.** Built a FastMCP server that Claude Desktop drives over stdio to generate product descriptions/summaries for all **882 products across 3 brands**, using product photos (raw image bytes) so descriptions reflect what's actually visible rather than hallucinated specs. `save_content` enforces a real content-quality gate — rejects entries over 80 words, containing raw measurements, or using banned marketing phrases. Iterated on prompt rules after catching the model inventing detail (e.g. calling plain-lens computer readers "blue-light blocking" just because the image looked that way).

**Frontend architecture decided:** backend/data-model-first, not frontend-first. Stack locked: Next.js App Router + Tailwind + shadcn/ui (chosen specifically because shadcn components are copied into the repo, fully ownable/customizable). Server Components + Suspense chosen over client-side fetching for SEO and progressive rendering; state kept in the URL rather than React state for shareability/crawlability.

*Reference docs written around this period (product-funnel.md, shape.md, ecommerce_rebuild_project_context.md, sellbrite_product_grouping_and_api_usage.md) captured the authoritative per-stage loss accounting for all three brands: ProSport 171→154 (17 lost), Monster 277→255 (22 lost), BikerShades 704→562 (142 lost, the heaviest loss rate — driven by ~30 Rx lens add-ons, admin/invoice line items, and duplicate-SKU products).*

### Phase 3 — Migration Completion → First Live Storefront + Payments (June 10–19, 2026)

## June 10–19, 2026

*Tail end of the WooCommerce → Supabase migration, followed by the first two weeks of building the real backend API, storefront frontend, and Stripe payments on top of it.*

### June 10 — Migration pipeline learnings (consolidated)

A written-up account of the full 17-step Python pipeline that moves all three brands from WooCommerce into Supabase. Key decisions documented here: single pipeline with brand injected at runtime; two slug fields per brand (pipeline key vs. DB/bucket slug — an early bug used the wrong one and sent images to the wrong bucket); per-brand scoped deletes (not `TRUNCATE`, after an early mistake wiped all three brands' data at once); no fallbacks in `import_items.py` (a `KeyError` signals incomplete upstream validation, not something to paper over); Veeqo always wins over WooCommerce for price/stock.

### June 11 — Frontend architecture planning (Next.js/React)

A "backend-first" architecture write-up. Locked in the storefront stack: Next.js App Router + Tailwind + shadcn/ui + Supabase + Vercel. Workflow order: data model → database → API contract → populate data → generate UI. URL-as-state discipline for SEO and shareability. Filtering/pagination pushed server-side, not done client-side over the full catalog.

### June 12 — Migration pipeline complete: final import numbers

The most complete account of the finished pipeline — this reads as the migration's actual completion milestone. **Final imported totals: 3 brands, 60 categories, 870 products, 3,108 product images, 9,633 variations, 2,085 variation images.**

Notable engineering stories: a mid-run crash left BikerShades partially imported, and the next delete hit an FK violation because live FKs lacked `ON DELETE CASCADE` — fixed with an explicit, ordered cascade of deletes as a belt-and-suspenders measure. Supabase's direct Postgres hostname resolves to IPv6, which corporate networks block outright — fixed by switching to the session pooler URL. Confirmed final gap: 1 Monster image + 78 BikerShades images genuinely 404 (not recoverable) — everything else accounted for.

### June 13 — Cart architecture & Next.js server/client model (conceptual)

Design notes for cart state: localStorage for guests, storing only `productId`/`variationId`/`quantity` — never price, stock, or images, which must always come fresh from the backend.

### June 14 — Migration refinements + first live API and frontend

**Milestone: first working backend + frontend.** Built a Next.js API (`sunglass-monster-server`, deployed to Vercel) serving public endpoints directly off the new Supabase schema, and a separate Next.js 16 frontend consuming it — homepage, category pages, product detail. This is the point where the migrated data became a browsable storefront for the first time.

### June 16 — Stripe decision, CORS/auth deep dive, and ProSport frontend build-out

**Stripe Checkout decision:** chose hosted Stripe Checkout Sessions over Payment Links or custom Stripe Elements — the hard parts (address validation, tax, 3D Secure, fraud, PCI compliance) are exactly what Stripe Checkout absorbs for free. **Actual ProSport frontend build:** category pages with two-phase streaming, slide-in header panels, auth via Supabase SSR server actions, and a session-refreshing middleware.

### June 17 — Auth flow hardening, cart/bookmark DB sync, legal checklist

Several sharp, specific bugs fixed: `onAuthStateChange` doesn't fire for server-action sign-in (solved with a manual `AuthProvider` context); a sign-out race condition from navigating away before awaiting `signOut()`; a cross-brand data pollution bug where the bookmarks endpoint was missing a brand filter, merging all three brands' bookmarks together. A broader legal/compliance checklist was drafted for launch (privacy policy, ToS, shipping/returns, accurate product claims, Stripe Tax).

### June 18 — Stripe payments shipped end-to-end (biggest day of this window)

The heaviest engineering day in this batch — Stripe Checkout went from decision to a fully working, idempotent payment pipeline. **Orders are created only in the webhook**, never at checkout-session creation. **Two layers of idempotency**: a Stripe-side idempotency key prevents duplicate sessions from double-clicks, and a DB-side unique constraint catches any webhook delivered twice. **Atomic sales counting via Postgres RPCs** avoids lost updates under concurrent webhook deliveries. **RLS design**: orders/order_items are select-only for authenticated users — the only write path is the webhook using the service-role key, making forged orders impossible via direct Supabase calls. A deliberate simplification: no stock gating at checkout — orders go through regardless, and the owner refunds manually if something can't be fulfilled.

### June 19 — Shipping addresses, variation-selector polish, migration pipeline v2

Stripe now collects shipping address during checkout, stored directly on the order — no shipping-address UI needed on the frontend at all. Variation selector UX rule: the primary attribute (e.g. color) must always show all options as available, even if the current secondary selection would normally rule some out. The migration pipeline was revised again (not just re-run): `brand_id` FKs dropped in favor of `brand_slug` text FKs everywhere; `stock`/`in_stock` removed from the schema entirely since Veeqo is the real source of truth for it.

**Overall arc of this window:** the migration pipeline reached a real, numbers-verified completion (870 products, three brands) by June 12, then got quietly revised again by June 19 as the schema simplified. In parallel, the team went from architecture notes to a fully working storefront with real Stripe payments, webhook-driven order creation, atomic inventory-sales counting, and shipping-address capture — essentially a functioning, launch-shaped e-commerce backend by the end of this ten-day stretch.

### Phase 4 — Parallel Backend/Frontend Build-Out (June 20–27, 2026)

## June 20–27, 2026

This stretch marks the real start of parallel backend/frontend build-out for the multi-brand platform. The backend track builds out the full public + authenticated API surface, database schema, Stripe integration, and hardens error handling day over day. The frontend track starts mid-week (June 24) and races to catch up, building the API client layer, providers, and page architecture on top of the backend contract.

### June 20 — Backend: API server v1 + OAuth/MCP research

First full pass at the Next.js API-only server: public catalog endpoints, authenticated user endpoints (cart, bookmarks, orders, checkout), and the Stripe webhook. A long side conversation worked through designing an OAuth/MCP authorization flow so Claude could call admin tools against the backend — exploratory/architectural, not yet implemented.

### June 21 — Backend: indexing and pagination deep dive

A dedicated study session on how Postgres pagination and indexing work under Supabase, producing a concrete recommended index set applied in the following days' migrations.

### June 22 — Backend: query rewrite + full endpoint pass

Rewrote the category endpoint to start from the junction table instead of `products`, matching the DB's cheapest query plan. Solidified the full 13-table schema and wrote the complete endpoint set with response envelope conventions.

### June 23 — Backend: two parallel sessions, product-detail correctness pass

Key decision, argued through carefully: public product URLs use `brandSlug + slug` for SEO-friendliness even though `id` would be the fastest lookup — solved with a composite unique constraint that's both a correctness guarantee and a performance index in one.

### June 24 — Backend + Frontend (first parallel day)

**Backend track:** consolidated the server — 13 endpoints deployed on Vercel, full test suite covering cart/bookmark/order/checkout cycles against the live deployment.

**Frontend track — genuine milestone: frontend build begins.** First frontend session, stood up the Next.js 16 App Router structure, the API client layer, and the URL/state design for product variant selection. Cart keyed by `productSlug:sku` end to end, matching the backend's composite-key convention — the two tracks converged cleanly on the same key format independently.

### June 25 — Backend: error-handling hardening; Frontend: error pages + type safety

**Backend:** systematic pass over error handling — every endpoint validates params first, checks DB errors explicitly, distinguishes "not found" from real failures. **Frontend:** built the four-way error-page taxonomy and hardened the API layer's TypeScript narrowing.

### June 26 — Backend: env var + auth cleanup; Frontend: three sessions building providers, brand system, homepage

**Frontend track — busiest day, three sessions.** Built the fetch-helper trio, the homepage architecture, made the brand slug public (a deliberate reversal from earlier server-only design), and built `AuthProvider` → `CartProvider` → `BookmarkProvider` as a nested context stack. Solved a genuinely interesting cross-cutting bug: a server component only knows auth state at request time, so a background cart-sync 401 mid-session could leave the header stale — fixed with a pattern that defers to the server's answer while a live context flag stays accurate afterward.

### June 27 — Frontend: provider architecture consolidated

Consolidating and documenting the provider system built the day before. By end of day the frontend's state-management foundation (auth, cart, bookmarks, error handling) was considered stable enough to build product/checkout pages on top of.

### Phase 5 — Admin Dashboard + Refunds + Design System (July 1–8, 2026)

## July 1–8, 2026

### July 1 — Frontend cleanup pass

A large refactor commit sweep across the storefront: centralized helpers, tightened types, a big header-icons rework, `ProductDetail.tsx` rewritten with a cleaner selection model, unavailable variant options now show a diagonal-line overlay instead of being disabled.

### July 2 — Backend API consolidation / Frontend polish

**Backend:** first full LEARNINGS write-up cataloguing the whole public + user API surface. Added a Postgres trigger to increment `total_sales`. **Frontend:** added a single random-products endpoint replacing ad-hoc category-iteration logic for homepage "Best Sellers."

### July 3 — Refunds, footer pages, and design milestone

**Backend:** added refund support end-to-end — `refunded_cents` column, a `charge.refunded` webhook handler, and a full order-status lifecycle. **Frontend:** shipped seven static footer pages (contact, FAQ, about, privacy, terms, shipping, returns).

**Design milestone — Admin Dashboard visual design system.** A static-HTML-mockup handoff doc establishing the admin's look before any component code was written: large serif display type for page titles paired with small-caps monospace for labels/breadcrumbs/table headers; an almost entirely black-and-white palette where color is reserved strictly for semantic meaning (green/red stock dots, red-outline sale badges); generous whitespace with thin-rule dividers instead of cards or shadows. Laid out the shared sidebar shell and specified four pages in detail — Orders, Products, Categories, Analytics.

### July 4 — Admin dashboard build begins

Built the actual admin app skeleton: `requireAdmin()` auth guard, two Supabase clients, the brand registry module, and the Overview page with three parallel stat queries. Added the `views` endpoint to atomically increment product/category view counts via Postgres RPCs.

### July 5 — Admin pages: Orders, Categories, Analytics

Built out three full admin pages. Categories: deliberately flattened the category tree into a sibling array because CSS Grid column alignment only works across direct siblings. Analytics: bar-chart fill is each item's *share of total*, not share of the leaderboard leader, so the visual represents true market share.

### July 6 — Admin Products page and SKU search

Products list required a two-part search because simple products store SKU on the product row while variable products store it per-variation — the query fans out to both tables in parallel and unions the results.

### July 7 — Refund system rebuild and order fulfillment

Reworked the order status model down to three states (processing, shipped, refunded), with partial refunds now derived rather than a separate enum value. Built a new DB trigger, `decrement_total_sales_on_refund`, with careful guard clauses to prevent double-decrementing on repeat Stripe events.

### July 8 — Navigation feedback and route cleanup

Added a slim animated top-of-page loading bar driven by navigation state, applied consistently across storefront and admin.

### Phase 6 — Admin Product Management + Try Before You Buy Launch (July 15–31, 2026)

## July 15–31, 2026

### July 15 — Backend: Admin product detail page (foundation)

Built the full create/edit product flow, backed by server actions and a client form covering simple vs. variable products, image uploads, color-picker attributes, category tags, and sale pricing. Product UUIDs are pre-generated client-side so images can upload before the product row exists in the DB.

### July 20 — Backend: Admin product page deepened

Formalized the simple-vs-variable data model and full client-side validation rules for save.

### July 22 — Backend: Description images & atomicity gap; Frontend: Auth flow, PKCE, and storefront housekeeping

Added a brand-level shared `description_images` table. Flagged a real known gap: product images/categories/description images use a delete-all-then-reinsert pattern on save that isn't atomic. **Frontend:** documented the full auth flow — sign-up, sign-in, forgot/reset password (PKCE-based), email confirmation — and switched to custom SMTP via Resend to get past Supabase's free-tier email rate limit.

### July 26 — Backend: Confirm-before-write UX + storage cleanup; Frontend: Auth hardening + schema documentation

Added a save/delete confirmation modal to guard against accidental writes. Slug handling became fully server-driven. **Frontend:** documented why `product_categories` is a join table rather than an array column — referential integrity, cascade deletes, bidirectional querying.

### July 28 — Backend: Admin form polish

Consolidated documentation of the confirmation modal, slug-sync banner, and attribute dropdown deduplication.

### July 29 — Frontend: Try Before You Buy — design phase

Pivoted to designing the "Try Before You Buy" (TBYB) feature — a BikerShades-only flow where customers try prescription frames at home before ordering lenses. Key architectural call: TBYB deliberately does not go through the existing cart, since adding nullable branching throughout cart code for one single-brand feature wasn't worth the complexity. Drafted two new tables: `tbyb_packages` (7 deposit-tier options) and `tbyb_submissions`.

### July 30 — Backend + Frontend: TBYB built end-to-end

**Backend:** built out the TBYB API — packages endpoint, file upload (decoupled from submission so files upload eagerly), and the submission endpoint. **Frontend:** shipped the TBYB page — a package grid followed by a 3-step form with dependent-field disabling. A clean sign of the feature graduating from stopgap to real implementation: the earlier direct-to-Supabase prototype was deleted now that the real backend endpoint existed.

### July 31 — Backend + Frontend: TBYB schema redesigned as a snapshot

The `tbyb_submissions` design changed from an FK-referenced schema to a fully denormalized one — submissions now store a full snapshot of the package captured at submission time, with no FK at all. Rationale: historical submissions must stay stable even if a package is later renamed, repriced, or deleted — a real "financial/fulfillment record" design tradeoff consciously made in favor of drift-proof history over referential purity.

### Phase 7 — TBYB Productionization (August 1–7, 2026)

## August 1–7, 2026

This week was almost entirely devoted to productionizing "Try Before You Buy": React port → Stripe checkout + idempotency → admin fulfillment tooling → account-page history → mobile/localStorage polish. A short side quest fixed a login-flash UI bug along the way.

### August 1 — TBYB: React port + backend endpoints

**Backend:** built the first three TBYB API routes. Key decision: all 28 submission columns are `NOT NULL`, with optional fields sent as the literal string `"None"` rather than `null`, so a missing field fails the insert atomically instead of writing a partial row. Locked down security: the `authenticated` role gets no INSERT grant on financial tables — a public anon key granting insert would let anyone skip the route and insert a spoofed price directly.

**Frontend:** ported the TBYB flow from a static HTML/vanilla-JS prototype into React, with upload-on-select instead of upload-at-submit.

### August 2 — Admin TBYB view + flow documentation

Built the admin TBYB table with filter tabs, expandable rows, and a draft/save status pattern. Seeded the seven real packages (BikerArmour, Wiley X, 7Eye, and their bundles, $229–$349 deposits).

### August 4 — Stripe checkout integration (milestone) + auth-flash bug fix

**Backend — the big lift this week:** replaced the simple "submit form → write a row" flow with a full Stripe checkout flow, snapshotting the package and hashing the entire form to prevent duplicates. The interesting engineering problem was duplicate prevention: a **partial unique index** on `form_hash` scoped to unpaid rows, plus handling a genuine race where two concurrent requests hash identically by catching the Postgres unique-violation and having the loser reuse the winner's ID.

**Bug fix — "The Auth Flash Problem":** a satisfying, well-diagnosed UI fix. Logged-in users briefly saw "Sign In" on every page load before it flipped to their profile icon. Root cause: a server-accurate boolean was combined with a client-side state that always started `false`. The fix changed that initial value from `false` to `null` ("not yet determined") and combined the two as `isSignedIn && (loggedIn ?? true)` — a one-line, low-risk change correctly modeling a genuine three-state system that had been wrongly collapsed into two.

### August 5 — TBYB state/upload redesign, localStorage refinement

Redesigned file upload to avoid a footgun where a `useEffect` watching form state could clobber a URL just restored from localStorage — fixed by pulling uploads out of state entirely and triggering them only from an explicit user action.

### August 6 — Admin fulfillment (shipping) + account-page history

Extended the admin TBYB table with a fulfillment block (carrier/tracking) with careful save/undo state logic keyed off persisted DB state, not the draft. Built the TBYB History section on the account page.

### August 7 — Final polish: success page, mobile uploads, saveToLS cleanup

Fixed a real mobile bug: Next.js server actions cap request bodies at 1MB by default, and mobile camera photos (3–10MB) were failing uploads silently — resolved by raising the body size limit to 10MB.

### Phase 8 — Rx Prescription Frames Feature (August 10–14, 2026)

## August 10–14, 2026

### August 10 — Backend: Stripe → Supabase → Veeqo order sync

Wired Veeqo inventory/fulfillment order creation into the Stripe checkout webhook. Key decision — **awaited sync, not background**: Vercel kills a serverless function the moment it responds, so any un-awaited work after `return` isn't reliable. Solved a genuine Veeqo double-POST race with an atomic conditional UPDATE that only lets one concurrent invocation "win" the right to sync. Found via testing that Veeqo's real API shape uses `sellables`, not `variants` as first assumed. End-to-end tests passed, including an order with deliberately different billing vs. shipping addresses.

### August 11 — Design: TBYB prescription-frame flow polish; Backend/data: prescription frames schema

**Design track:** refinements to the prescription frame selection flow — frame browsing, color swatches, visible Rx range per frame, a running selection summary, and a formalized 5-step prescription entry flow. Fixed a real bug: PD required-field validation had been wired to the wrong step, letting a customer advance without entering required values.

**Backend/data track:** defined the JSON schema and content rules for the prescription-frames dataset ahead of migration. At this point: 78 frames across four brands.

### August 12 — Backend / RX-frames migration: full data pipeline shipped

The prescription-frames catalog moved from flat-file staging to a live database-backed API in one session, ending at **86 frames across 4 brands**. **Deliberately manual, not scraped**: each frame was entered by hand from a screenshot of the vendor product page, because Rx-specific details needed human judgment a scraper would get wrong — two frames were dropped entirely because their pages never stated an Rx range, following a "never assume, flag and skip" rule. Built a 16-check Python validator and an idempotent migration script. Final migration run: **86/86 migrated** successfully.

### August 13 — Frontend: PrescriptionFramesClient build

Built the client component powering the Rx frame ordering flow — a frame-grid-to-checkout flow with a 7-step form, file uploads, and localStorage persistence. Careful state design separating "which frame's form is open" from "which swatch is clicked" from "mouse-over display only," with selected color intentionally *derived* rather than stored to avoid state-sync bugs.

### August 14 — Backend: Rx order endpoint, deposits, and idempotency

Built the server side the frontend was waiting on. New table stores a full snapshot of the order rather than an FK back to the live catalog. **Deposit mechanics:** a TBYB purchase's deposit can be applied to an Rx order, computed at read time. **Double-spend prevention on the deposit** was the hard part, worked through in three iterations of session-claiming logic until concurrent requests were provably safe — a genuinely intricate piece of engineering for a small feature.

### Phase 9 — Final Polish + Production Launch (August 16–24, 2026)

## August 16–24, 2026

This batch covers the final stretch of the summer project: finishing the Rx order flow, a round of account/footer/legal polish, a cluster of Stripe correctness fixes, and — starting August 22 — the actual production cutover: real domains moved off WordPress/SiteGround onto Vercel, transactional email reconfigured, Stripe flipped from test to live mode, and a full clean production data migration. It closes on August 24 with a staging environment stood up as a mirror of production.

### August 16 — Rx Orders: schema and checkout flow built

First full build of the prescription-frame order flow, BikerShades-only. Pricing computed server-side from hardcoded price dicts so the client can never manipulate cost. A 6-step form with localStorage persistence per brand/step.

### August 17–18 — Rx Orders: admin UI and hardening

Built out the admin side mirroring the existing TBYB admin pattern. Refined frame-color selection so localStorage only ever holds a complete pair once the user explicitly commits — eliminating a class of partially-selected-state bugs on refresh.

### August 19 — Rx order history on the account page

Added customer-facing Rx order history, mirroring the existing TBYB submissions component.

### August 20 — Delete Account, Rx form bug fixes, footer/legal copy

Added a full delete-account flow. Fixed two real bugs: an overzealous auth guard redirecting users before they'd even started a form, and a localStorage bug that was stripping name/email on save.

### August 21 — Stripe correctness pass; SQL/schema review

A cluster of subtle Stripe and data-integrity bugs found and fixed, including a bug where an expired Stripe session could be handed back to a user resubmitting a form, and `customer_email` using an unverified form-submitted address instead of the authenticated user's actual email.

### August 22 — Auth-gating, image pipeline switch, and the start of the domain migration

**Migrated all `next/image` usage (13 files) to native `<img loading="lazy">`** — Vercel's image-optimization free tier was getting exceeded across all three brand deployments, causing images to fail and show alt text.

**Milestone — domain migration begins:** a full session moving all three sites from SiteGround/WordPress to Vercel with real custom domains — DNS record migration (A/MX/CNAME/TXT, SPF/DKIM/DMARC), Resend transactional email setup, and rebuilt branded HTML auth email templates. Found and fixed a bug where confirmation-email links were falling back to the raw `.vercel.app` preview URL instead of the real domain — a bug that would have broken signup confirmation and password reset in production had it shipped.

### August 23 — Going-live day: DB hardening, Stripe live mode, maintenance-mode switch, and a full production data reset

**The most consequential day in the whole project**, spanning four tracks: schema hardening and RLS enabled across all catalog tables; a documented "Going Live" checklist for switching Stripe to live mode; a **maintenance-mode kill switch** built specifically to allow safely taking a site down during the domain cutover; and — the centerpiece — a **full production database and Storage reset with a clean reimport of all three brands' catalogs**, surviving a real incident along the way (all three original WooCommerce source sites had become partially unreachable since the original scrape, with zero fallback — recovered by re-deriving every image's filename deterministically and re-downloading it from Supabase Storage itself instead of the dead source). Final state, verified against source data with **zero mismatches across every table for all three brands.**

### August 24 — Staging environment stood up as a mirror of production

A second, independent Supabase project created and populated as a staging environment, following the same pipeline used for production the day before. Caught three environment-hygiene bugs along the way. Final verification shows an exact match with production across every table, confirming staging is a true, independent mirror.

**How the documented project ends:** The story arc closes here at go-live. There's no explicit "we shipped it" statement in the files, but the sequence — domain cutover, live payments, full data reset, staging established — reads unambiguously as the production launch, consistent with the dated production screenshots (`prod-bikershades-landing-page.png` and others) saved for this window.

---

## Photo Manifest

*All screenshots below live in the same source folder. Grouped by what they show; suggested section is a starting guess, not final.*

### Production launch screenshots (strongest hero/gallery material — the finished, live site)
- `prod-bikershades-landing-page.png` — BikerShades storefront homepage
- `prod-prosport-landing-page.png` — ProSport storefront homepage
- `prod-sunglass-monster-landing-page.png` — Sunglass Monster storefront homepage
- `prod-product-page.png` — product detail page
- `prod-bag-panel.png` — cart/bag slide-in panel
- `prod-checkout-page.png` — checkout page
- `prod-stripe-checkout.png` — Stripe hosted checkout
- `prod-account-page.png` — customer account page
- `prod-tbyb-page.png` / `prod-tbyb-form.png` — Try Before You Buy flow, live
- `prod-rx-frames-page.png` / `prod-rx-frames-form.png` — Rx Frame ordering flow, live
- `prod-admin-overview-page.png` — admin dashboard overview
- `prod-admin-orders-page.png` — admin orders
- `prod-admin-products-page.png` — admin products
- `prod-admin-categories-page.png` — admin categories
- `prod-admin-analytics-page.png` — admin analytics

### Admin dashboard — early build/design
- `admin-overview.png`, `admin-orders.png`, `admin-sign-in.png` — earlier-stage admin screenshots (pre-production, useful for a "before/after" comparison against the `prod-admin-*` set)
- `claude-design-admin-tbyb.png` — design mockup for the admin TBYB view

### Claude Design mockups (design-phase artifacts, good for a "how it was designed" section)
- `claude-design-frontend-landing.png`
- `claude-design-frontend-product.png`
- `claude-design-frontend-products.png`
- `claude-design-frontend-rx-frames.png`
- `claude-design-email-confirmation-template.png`

### Database / migration pipeline
- `database-001-004-overview.png`, `database-001-004-overview-cropped.png`, `database-001-005-overview.png`, `database-overview-001-003.png` — schema/migration overview screenshots
- `database-cart-bookmarks-tables.png` — cart/bookmarks schema
- `database-prod-001.png` through `database-prod-005.png` — production database state (go-live verification)
- `database-rx-frames-addition.png` — Rx frames table addition
- `migration-mcp-prompt.png`, `migration-mcp-server.png` — the AI content-generation MCP server in action
- `migration-prod-database-seed.png` — production seeding

### Veeqo / inventory integration
- `veeqo-and-old-site-issue.png`
- `veeqo-endpoint-curl-testing.png`
- `veeqo-order-creation-200.png`, `veeqo-order-creation-success.png` — successful order sync tests
- `veeqo-rest-api-documentation.png`

### Frontend feature screenshots (dev-time, not final production)
- `frontend-bookmarks-local-storage.png`
- `frontend-faq-page.png`
- `frontend-prod-503-maintainence-redirect.png` — the maintenance-mode kill switch in action
- `frontend-rx-frames-page.png`

### Payments / infrastructure setup
- `stripe-payment-setup-call.png`
- `resend-smtp-provider.png`

### Branding
- `nano-banana-new-sunglass-monster-logo.png` — Sunglass Monster logo exploration
- `sunglass-monster-compressed.png`

### Reference / before
- `old-site-rx-frame-page.png` — the legacy WooCommerce Rx frame page (good "before" shot for a before/after comparison)

### Dev tooling
- `backend-nextjs-debugger.png`

---

*End of source document. Next step: hand the Six Did/Learned Threads section to Claude Design as the backbone, using the Timeline and Photo Manifest as backup detail — not to build a case study, but a "what I did / what I learned" page.*
