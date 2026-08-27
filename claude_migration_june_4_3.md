# Migration Session Learnings (June 4, 2026)
_Session: 2026-06-05_

---

## What we built

### `pipeline-shapes.md`
A reference doc at the project root covering the full data journey:
- Slim product shape (WooCommerce admin API fields)
- Slim variation shape (singular `image` object, not array)
- All validation rules at each stage in table form
- All reshape transformations (field renames, price → cents, HTML strip, etc.)
- Final create-items shape with embedded variations

### Base SKU mismatch check — C-prefix rule
Updated `validate_variations.py` to allow a single leading `C` difference between pre-dash SKU bases:

```python
normalized = {b[1:] if b.startswith("C") else b for b in bases}
if len(normalized) > 1:
    # still flag
```

`CPR29X36ST` and `PR29X36ST` normalize to the same base → allowed.  
`CA10X15CP` and `CA10X16CP` → `A10X15CP` vs `A10X16CP` → still flagged.  
`CAPL` vs `CA` → `APL` vs `A` → still flagged.

This unblocked BikerShades combo products that were incorrectly excluded before (+12 products, +437 variations).

---

## What we learned

### Veeqo export
- **What it is:** Sellable items export from Veeqo (inventory/channel management platform). One row per SKU, includes product title, variant title, price, weight, dims, qty_on_hand, UPC.
- **Coverage:** BikerShades 99%+, Monster 98%, ProSport 10% (ProSport not managed in Veeqo).
- **What it's good for:** SKU validation only — confirming that a SKU exists in active inventory management. Not useful for data enrichment.
- **What it's NOT good for:**
  - `qty_on_hand` — only 8% of rows have any value. Veeqo is not tracking live stock for most SKUs. Use WooCommerce `stock_quantity`.
  - `sales_price` — 79% mismatch with WooCommerce. Veeqo stores channel-specific prices (Amazon, eBay), not store prices. WooCommerce is authoritative.
  - `cost_price` — 7 rows populated. Basically empty.
- **UPC codes** — 20% coverage. Only field Veeqo has that WooCommerce doesn't. Worth keeping if barcodes ever matter.

### Combo product SKU pattern
BikerShades and ProSport have "combo products" — bundles of multiple individual products sold together. Their variations have a `C` prepended to the regular SKU base: `CPR29X36ST` is the combo version of `PR29X36ST`. WooCommerce groups combo and non-combo variations in the same parent product. The `C` prefix is the distinguishing pattern — not a data error.

### Pipeline funnel (final accurate counts after C-prefix fix)

| Stage | ProSport | Monster | BikerShades |
|-------|----------|---------|-------------|
| Fetched | 171 | 277 | 704 |
| validate-products | 169 (-2) | 267 (-10) | 611 (-93) |
| create-items | 161 (-8) | 260 (-7) | 565 (-46) |
| Veeqo matched | 154 (-7) | 255 (-5) | 562 (-3) |

**Where products drop:**
- validate-products: private/draft status, missing SKU, no images, attribute name collisions
- create-items: all variations excluded at validate (base SKU mismatch), duplicate SKUs
- Veeqo: readers/bifocals not synced to Veeqo (website-only products)

BikerShades's 93 validate-products losses are mostly WooCommerce junk — Rx add-ons, private listings, admin products. ProSport and Monster are much cleaner.

### Veeqo as a data source — what to expose
The only actionable use case is the SKU match check as a final validation step before import. Products whose sellable SKUs all appear in Veeqo = confirmed active inventory. Those that don't = website-only or inactive — flag for review but not a pipeline blocker.

---

## Key pipeline state after this session

- `validate_variations.py` — C-prefix normalization added to base SKU mismatch check
- `pipeline-shapes.md` — new reference doc documenting full data shapes and all checks/transforms
- All three brands re-run through validate_variations → reshape_variations → create_items with updated check
- BikerShades gained 12 products (combo products unblocked)
- ProSport/Monster slightly lower than old baselines (genuine mixed-model groupings now correctly excluded)

---

## Next steps (unchanged)
- Write `bikershades_categorize_items.py`
- Write `monster_categorize_items.py`
- ProSport run-2: download → upload → import
