# Migration Pipeline — Session Notes (June 2, 2026)
_2026-06-02_

---

## What was built

### Pipeline scripts (new-migration/pipeline/)

| Script | What it does |
|---|---|
| `reshape_products.py` | Normalizes 169 validated products: strips HTML, prices to cents, merges stock, cleans attribute names, always-object dimensions |
| `reshape_variations.py` | Normalizes 1,740 validated variations: same conventions + Power option normalization, grouped structure preserved |
| `create_items.py` | Embeds variations into products with triple ownership check, attribute expansion, image dedup, min/max price → 166 items passed, 3 flagged |

### QA scripts (new-migration/qa/)

| Script | What it checks |
|---|---|
| `audit_reshape_products.py` | Field coverage, pricing stats, attribute breakdown |
| `verify_reshape_products.py` | Cross-checks reshape-products vs validate-products |
| `audit_reshape_variations.py` | Field coverage, pricing, Power normalization results |
| `verify_reshape_variations.py` | Cross-checks reshape-variations vs validate-variations |
| `audit_create_items.py` | Pricing ranges, image dedup stats, attribute/variation counts |
| `verify_create_items.py` | Full ownership, dedup, price, SKU uniqueness checks — PASSED |

### Docs updated

- `docs/reshape-products/` — notes, audit, verify
- `docs/reshape-variations/` — notes, audit, verify
- `docs/create-items/` — notes, audit, verify
- Root `CLAUDE.md`, `PROGRESS.md`, `README.md` — all reflect new script/folder names

---

## Key decisions and conventions

### Nullability rules
- Empty strings → `null` for scalars
- `[]` (empty array, never `null`) for: `categories`, `images`, `attributes`, `variations`, `descriptionImages`, `shortDescription`
- `dimensions` is the one exception — always an object `{length, width, height}` even when all fields are null, because the shape is always the same
- Follows WooCommerce's own null practices except empty strings

### Prices
- All prices stored as integers in cents: `round(float(val) * 100)`
- Variable product `minPriceCents` / `maxPriceCents` derived from variation `regularPriceCents` (the stable "was" price, not the current selling price — so the range doesn't shift when sales toggle)
- `salePriceCents` is per-variation only; never set at the product level

### Attribute name cleaning
- Products: strip `"Choose "` prefix **and** `" for Filter"` suffix
- Variations: strip `"Choose "` prefix only
- Applied in `clean_attr_name()` helper in both reshape scripts

### Power option normalization
- WooCommerce stored bifocal power values inconsistently: `1-5`, `2`, `2-5` instead of `1.50`, `2.00`, `2.50`
- Fix: `f"{float(val.replace('-', '.')):.2f}"` — handles hyphens as decimal points, forces 2 decimal places
- Applied to any attribute whose cleaned name is `"Power"` in both products and variations

### create_items — ownership triple-check
A variation is only embedded into a product if:
1. `variation.id` matches the ID slot in `product.variations[]`
2. `variation.parent_id` matches `product.id`
3. The variation came from the correct `product_id` entry in reshape-variations (implicit, from grouped structure)

### Attribute expansion (not flagging)
If a variation uses an option that the parent attribute doesn't list, the option is added to the parent. This fixes WooCommerce data where parent attributes weren't updated when new variations were added.  
Rule: variation attribute **name** must match a parent attribute name — if the name itself is absent, that is flagged.

### Image dedup
If a variation's `image.src` matches any `src` in the parent's `images[]`, `image` is set to `null`.  
Result: 1677/1713 variations (97.9%) had their image nulled. Only 36 carry a unique image.

---

## Key results (ProSport)

| Stage | In | Out | Flagged |
|---|---|---|---|
| validate_products | 171 | 169 | 2 (draft copy, no-SKU simple) |
| validate_variations | 1,758 | 1,740 | 18 (17 SKU-spaces, 1 no-price) |
| reshape_products | 169 | 169 | — |
| reshape_variations | 1,740 | 1,740 | — |
| create_items | 169 | 166 | 3 (unreplaced variation IDs from the 18 flagged above) |

The 3 flagged items (2145, 2211, 1208) are a direct downstream consequence of the 18 flagged variations — not new issues. Fix those 18 in WooCommerce and re-run.

---

## What's next

### Recategorize
21 products have only `"Uncategorized"` as their category. Proposed assignments (hardcoded per brand):

**Sunglasses → assign a real category:**

| Product | Category |
|---|---|
| Apollo Safety Wrap (has Power attr) | FULL LENS |
| Yogi Mirror Aviator | METAL & AVIATORS |
| Mako Metal Mirror | METAL & AVIATORS |
| Zulu Metal Fashion | METAL & AVIATORS |
| Brandy | METAL & AVIATORS |
| Carmen Fashion | STREET |
| Popcat | STREET |
| Miami Square Fashion | STREET |
| proSPORT Kuna | STREET |

**Accessories/cases (12) → `categories: []` (no category):**  
Leopard/Ruby/Jade/Garnet/Diamond/Ametrine/Hematite/Citrine/Agate cases + Repair Kit + Screw Driver + Lens Cleaner

**Other recategorize operations (all products):**
- Drop `"Uncategorized"` from every product that has it
- Decode HTML entities: `&amp;` → `&` in category names
- Rename `METALS & AVIATORS` → `METAL & AVIATORS`

### After recategorize
- `download_images.py` — crash-safe via `url_map.json`
- `upload_images.py` — crash-safe via `upload_map.json`
- `import_items.py` — insert into Supabase DB

---

## Folder/naming conventions established

| Old name | New name |
|---|---|
| `validated-products/` | `validate-products/` |
| `validated-variations/` | `validate-variations/` |
| `reshaped/` | `reshape-products/` |
| `reshape_items.py` | `reshape_products.py` |
| `combine_items.py` | `create_items.py` |

All output scoped to `data/{brand_slug}/` — brands never collide on disk.

---

## Bug caught and fixed

**`<img>` tag fragments in product descriptions**  
Initial regex only removed the `src` URL portion, leaving ` alt="" width="700" height="350">` dangling as text. Fixed with a two-pass approach: full `<img[^>]*>` removal for cleanup, then `src` extraction for `descriptionImages`.
