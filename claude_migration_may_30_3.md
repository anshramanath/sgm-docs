# Pipeline Design & Decisions (May 30, 2026)
_Last updated: 2026-05-30_

## What this is

Migrating 647 WooCommerce products from bikershades.com into a new multi-brand Supabase-backed app. Scripts live in `migration/pipeline/`. Run them in order from the repo root.

---

## Pipeline

| Step | Script | Input | Output |
|------|--------|-------|--------|
| 1 | `fetch_products.py` | WooCommerce API | `items/products.json` (647 products) |
| 2 | `fetch_variations.py` | `products.json` | `items/variations/variations.json` (6,064 variations) |
| 3 | `reshape_items.py` | `products.json` + `variations.json` | `reshaped/*.json` (8 buckets) |
| 4 | `recategorize_items.py` | `reshaped/shaped_items.json` | `recategorized/*.json` |
| 5 | `download_images.py` | `recategorized/categorized_items.json` | `images/local/`, `images/url_map.json` |
| 6 | `upload_images.py` | `images/local/`, `url_map.json` | `images/upload_map.json` |
| 7 | `import_items.py` | `categorized_items.json` + both maps | Supabase DB |

Each script prints QA reminders at the end. Run the listed QA scripts after each step.

---

## Reshape buckets (step 3)

Products split by issue type. Only `shaped_items.json` proceeds through the pipeline. Others are held for owner review.

| File | Count | Reason |
|------|------:|-------|
| `shaped_items.json` | 586 | Clean — ready for recategorization |
| `invalid_sku_items.json` | 22 | SKU missing or contains spaces |
| `no_weight_items.json` | 14 | Weight null or 0.0 |
| `zero_price_items.json` | 8 | Regular price is zero |
| `missing_attributes_items.json` | 7 | Variation attribute data missing from API |
| `duplicate_name_items.json` | 6 | Two products share the same reshaped name |
| `no_image_items.json` | 3 | No product images at all |
| `multiple_variation_images_items.json` | 1 | Variation has >1 image (WooCommerce misconfiguration) |

---

## Recategorize buckets (step 4)

| File | Count | Reason |
|------|------:|-------|
| `categorized_items.json` | 581 | Ready for import |
| `no_category_items.json` | 2 | All their category links were dropped |
| `single_leaf_items.json` | 3 | Only item in their leaf category |

### Category format

Categories stored as `[[{level, name, slug}]]` — an array of root-to-leaf paths. Products only land at leaf nodes. Max depth is 2 hops from root to leaf.

```json
"categories": [
  [
    {"level": 1, "slug": "sunglasses", "name": "Sunglasses"},
    {"level": 2, "slug": "polarized", "name": "Polarized"}
  ]
]
```

### Recategorization rules (applied in order)

- **Drop** — remove the link entirely (e.g. `all-products`, `see-all-brands`)
- **Merge** — remap slug to canonical slug (e.g. `mirrored` → `mirror`)
- **Collapse** — single-item leaf reassigned to parent
- **Strip** — remove non-leaf paths (products must only sit at leaves)
- **Single-leaf removal** — products that end up as sole occupant of a leaf are moved to `single_leaf_items.json`

---

## Image pipeline (steps 5–6)

### Naming scheme

Images are named using the full path segment after `/uploads/` in the URL, with `/` replaced by `_`:

```
https://bikershades.com/wp-content/uploads/2024/05/Hangar-Gunmetal-Front.jpg
                                            └─────────────────────────────┘
                                            → 2024_05_Hangar-Gunmetal-Front.jpg
```

This makes every filename globally unique — no two URLs sharing the same date-prefixed path — eliminating collisions entirely.

### Maps

Two JSON checkpoint files act as the source of truth. The pipeline never re-downloads or re-uploads what's already in the maps.

```
url_map.json    { "https://bikershades.com/..." → "/abs/path/local/image.jpg" }
upload_map.json { "/abs/path/local/image.jpg"   → "https://supabase.../image.jpg" }
```

### Resolution chain at import

```
bikershades URL → url_map → local path → upload_map → Supabase URL
```

If either lookup fails, the image is dropped. No broken URLs are inserted into the DB.

### failed.json

```json
{
  "failed_to_download": [{"sku": "BK-123", "src": "https://bikershades.com/..."}],
  "collisions": [
    {
      "existing": {"src": "https://...", "sku": "SKU-A"},
      "tried":    {"src": "https://...", "sku": "SKU-B"}
    }
  ]
}
```

- `failed_to_download` — URLs that returned non-200 or errored. Cleared on successful retry.
- `collisions` — filenames that conflicted (should be empty with the new naming scheme).

Both maps are crash-safe: checkpointed after every individual download/upload.

---

## Database schema

Multi-brand Postgres schema on Supabase. Auth via Supabase Auth, admin access gated by `admins` table. Frontend never hits Supabase directly — goes through Next.js backend with `service_role` key.

| Table | Key points |
|-------|-----------|
| `brands` | `slug` matches the Supabase Storage bucket name |
| `categories` | Self-referential `parent_id` tree + `slug` for URL routing |
| `product_categories` | Junction table — product to leaf category |
| `products` | `attributes` is `jsonb` (`[{name, terms[]}]`) for listing-page queries without joining variations |
| `variations` | `variation text[]` — flat slug array e.g. `["black-clear", "1.00"]`. One image max, stored inline (`image_src`, `image_name`) |
| `product_images` | Multiple images per product, ordered by `sort_order` |
| `description_images` | Images extracted from description HTML; belong to either a product or variation (enforced by check constraint) |

Source of truth: `migration/db/001_initial_schema.sql`

---

## QA system

Every pipeline step has a corresponding audit and verify script in `migration/qa/`.

| Step | Audit | Verify |
|------|-------|--------|
| Reshape | `audit_shaped_items.py` | `verify_shaped_items.py` |
| Recategorize | `audit_categorized_items.py` | `verify_categorized_items.py` |
| Download | `audit_downloaded_images.py` | `verify_downloaded_images.py` |
| Import | `audit_import.py` | `verify_import.py` |

- **Audit** — checks the output's own internal validity (field presence, format, counts)
- **Verify** — cross-references output against input to confirm nothing was lost or corrupted

Results go in `docs/*/audit.md` and `docs/*/verify.md`. Each QA script reminds you to update those files.

---

## Gitignore

Data directories are gitignored — only scripts, schema, docs, and config are committed.

```
migration/items/         ← raw WooCommerce fetch output
migration/reshaped/      ← reshape output
migration/recategorized/ ← recategorize output
migration/images/        ← downloaded images + maps
```
