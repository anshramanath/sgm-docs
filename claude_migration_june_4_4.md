# Session Learnings (June 4th, 2026)
_WooCommerce → Supabase migration pipeline — built and refined across multiple sessions_

---

## What we built

### Pipeline scripts added
- **`pipeline/create_items.py`** — non-product SKU filtering via `JUNK_SKU_PREFIXES`, `JUNK_SKU_SUFFIXES`, `JUNK_SKU_CONTAINS` dicts (per brand). BikerShades catches USPS labels (`LABEL-`), Try Before You Buy deposits (`TRYB4UB-`), and Rx frame listings (`-RX` token anywhere in SKU).
- **`pipeline/pipeline-shapes.md`** — reference doc covering slim product shape, slim variation shape, all validation rules, reshape transformations, and final create-items shape.

### Analysis scripts added
- **`analysis/analyze_veeqo_items.py`** — Veeqo SKU coverage report: passed overview, per-flagged-product detail with missing %, power breakdown, aggregate stats.

### QA scripts added
- **`qa/veeqo_items.py`** — informational Veeqo SKU check. Reads `create-items/items.json`, prints which items have SKUs absent from Veeqo, writes no files. Does not gate the pipeline.

### Docs added
- **`new-migration/product-funnel.md`** — full per-stage funnel counts for all three brands, flagging rationale at every stage, cross-brand patterns, Veeqo findings, questionable flags, BikerShades features to build, open questions.

### Pipeline fixes
- **C-prefix base SKU rule** — `CPR...` vs `PR...` is a combo vs single version of the same product; allowed through the base SKU mismatch check. Previously flagged 12 BikerShades products (+437 variations) incorrectly.
- **`-RX` token check** — `7EYE-DIABLO-RX-HIGH` slipped through the original `-RX` suffix check because the suffix was `-HIGH`. Fixed with a `JUNK_SKU_CONTAINS` check for `-RX` as a substring.

---

## What we learned

### About the data

**SKU base mismatch is the dominant cause of variation loss.** Three root causes across all brands:
1. **Polarized vs non-polarized mixed into one WC product** — same frame, different lens treatment, inconsistent SKU prefix (e.g. `SP73X31EL` vs `SPPL73X31EL`).
2. **SKU revision coexisting with original** — product re-SKU'd but old and new variations both live in the same WC listing (e.g. `BF4X33FL` vs `BF4X33FLCT`, `CA25X095MR` vs `CA25X095MRMIRROR`).
3. **Multiple distinct models consolidated under one WC product** — most visible in BikerShades' BikerArmour goggle lines (Socket, Magnum, Evolution, Cayman, Anaconda, Angel), each sold separately by lens type but also appearing in a consolidated listing with mixed SKU bases. Harvard Multifocal Reader has the same issue.

**Bifocals and readers dominate Veeqo misses.** 13 of 15 Veeqo-absent products across all three brands are bifocals or readers. Two sub-patterns:
- Entire product absent from Veeqo (likely discontinued or never synced).
- High-power variations missing (2.25+ diopter SKUs never added to inventory) even when lower powers are present.

**BikerShades has a large non-product WooCommerce tail.** 93 products flagged at validate_products — Rx add-on services, private/draft entries, admin placeholders, accessories with no SKU. These drive the high loss rate (35% total). 150 more flagged at create_items including 94 Rx frame listings and 7 Try Before You Buy deposits that need to be rebuilt as features.

**Variable product stock is null by design.** WooCommerce aggregate stock on variable products is misleading and often wrong. Variation-level stock is the source of truth.

**All-or-nothing dimensions.** Partial dimension data (e.g. length + width but no height) is silently wrong. Pipeline nullifies all three if any are missing.

### About Veeqo

**Veeqo is downstream of WooCommerce.** WooCommerce feeds Veeqo as a sales channel. This means:
- SKUs missing from Veeqo could be absent for many reasons unrelated to product validity.
- Veeqo cannot be used to gate which products move forward in the pipeline.
- `qa/veeqo_items.py` is purely informational.

**Veeqo prices are unreliable.** 431 SKUs appear twice in the Veeqo export (native entry + WC channel-synced copy), 162 with conflicting prices. No way to identify which is the master. WooCommerce prices are the correct baseline.

**Veeqo descriptions were updated independently.** ~241 products have only raw dimension specs in WooCommerce but full marketing copy in Veeqo. The two description sources diverged. Decision: do not use Veeqo descriptions — same reliability concern as prices, WooCommerce is authoritative.

**Veeqo stock is empty** in the export. Not useful.

### About the pipeline design

**False positives are preferable to false negatives.** Better to flag too much and review manually than to let bad data through silently. Professor Multifocal Reader (1 bad variation out of 24) is correctly excluded even though 23 variations are valid — the pipeline's all-or-nothing flag rule is a feature, not a bug.

**`any flag = full exclusion` at create_items is intentional.** A product with unreplaced variation IDs has WooCommerce data issues that should be reviewed. Passing it through partially would mask the upstream problem.

**The Veeqo SKU check caught a junk product that SKU patterns missed** — `7EYE-DIABLO-RX-HIGH` had a non-standard Rx SKU format and was in Veeqo (so it wouldn't have been caught by the Veeqo check either). Only caught by manual review of `create-items/items.json`. Lesson: review the final passed item list, not just the flagged list.

### About BikerShades specifically

**Try Before You Buy and Rx frame submission** were implemented as individual WooCommerce product listings (100 products total). These should be rebuilt as first-class features on the new store — a single TBYB program page and a single Rx submission flow — rather than migrated as products.

**Wiley X and 7eye/Ziena frames** are stocked and sold as both regular sunglasses (variable products with color/lens variations) and as Rx frame submissions (simple products with `-RX` SKU). The Rx listings are flagged and excluded; the regular variable product listings pass through normally.

---

## Final create-items counts

| Brand | Items | Variations | Flagged |
|---|---|---|---|
| ProSport | 161 | 1,646 | 8 |
| Monster | 260 | 2,537 | 7 |
| BikerShades | 461 | 5,239 | 150 |

All audit + verify scripts passed for all three brands.

---

## Next steps

1. Write `monster_categorize_items.py`
2. Write `bikershades_categorize_items.py`
3. ProSport: `download_items.py` → `upload_items.py` → `import_items.py`
4. Monster + BikerShades: categorize → download → upload → import

