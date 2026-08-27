# Product Funnel (June 4, 2026)
_WooCommerce fetch → Veeqo match_

---

## Summary

|  | ProSport | Monster | BikerShades |
|--|----------|---------|-------------|
| Fetched | 171 | 277 | 704 |
| After validate | 169 (-2) | 267 (-10) | 611 (-93) |
| After create-items | 161 (-8) | 260 (-7) | 565 (-46) |
| After Veeqo match | 154 (-7) | 255 (-5) | 562 (-3) |
| **Total lost** | **17** | **22** | **142** |

---

## Stage 1 → validate-products

Drops: private/draft status, missing SKU, no images, attribute name collisions, missing price.

**ProSport (-2)**
- Cabriolet Mirror Bifocal (Copy) — draft, no slug
- Sunglass String Chum Strap — no SKU

**Monster (-10)**
- Holiday Reader, Dorothy Reader — attribute name collision (`Lens Type for Filter` + `Filter by Lens Type` both clean to `Lens Type`)
- Socket Goggle, Childrens Boys Girls Flip Up Shield, Sunglass String Chum Strap, VShield Bandannas, Black Face Mask — no SKU
- proSPORT VShield Face Shield — private + SKU spaces + attribute collision
- proSPORT Rocket High Wrap Reader, Leopard Print Hard Case — no images

**BikerShades (-93)**
Mostly WooCommerce back-office junk never meant to be sold as real products:
- ~30 Rx Options (lens add-ons: A/R, anti-fog, transitions, mirror coatings, remakes) — private, no SKU, no images
- Draft/copy products — no slug, draft status
- Admin line items (Distributor Invoice, Misc Shipping Cost, Custom Product, Expedited Shipping) — no SKU, no price
- Accessory parts (Wiley X Strap, Cleaning Cloth, Replacement Foam, Nose Pieces) — SKU spaces or no images
- Discontinued/pending frames (Pike Polarized, Wiley X Compass, BikerArmour Trooper) — private or pending
- 1 product with unsupported type (`prescription` instead of `variable`/`simple`)
- 2 attribute name collisions (7eye Warrior for Eclipse XT)

---

## Stage 2 → create-items

Drops: all variations excluded at validate-variations (base SKU mismatch), duplicate SKUs.

All "no filled variations" cases = the product passed validate-products but every variation was kicked out at validate-variations for base SKU mismatch (two different model numbers grouped in one WooCommerce product).

**ProSport (-8)**
- Sharkbait Polarized Float, Chipper Aviator — variations all have base SKU mismatch
- Professor Multifocal Reader — unreplaced variation IDs (variations not in validated set)
- Eliminator Polarized Bifocal, First Lady Bifocal, Kumatage Bifocal, Cooper Bifocal, Morocco Mirror — base SKU mismatch (`CA25X095MR` vs `CA25X095MRMIRROR`, `FLPL7X59SB` vs `FLPL73X31SB`, etc.)

**Monster (-7)**
Same pattern — all proSPORT-branded products listed on Monster with mixed variation groupings:
- Sharkbait, Chipper, Professor, Harvard, First Lady, Kumatage, Morocco Mirror

**BikerShades (-46)**
- 44 products with no filled variations — all had their variations excluded for base SKU mismatch (polarized `CAPL` vs non-polarized `CA` prefix mixed in one WooCommerce product, or other model mixing)
- 2 duplicate SKUs: warranty product (62556, `LABEL-USPSRETURN-1-1`) and face masks (49895, `clothmasks`)

---

## Stage 3 → Veeqo match

Drops: products whose sellable SKUs don't appear in Veeqo. These are real valid products that just weren't synced to Veeqo — website-only or inactive channel listings.

13 of 15 unmatched are bifocals or readers. Readers/bifocals have too many power-specific SKU combinations (1.00–6.00+) to manage well on marketplace channels — likely sold website-only. Only 2 exceptions: Mysto Sunglasses and Nautica Polarized (both Monster), which just weren't synced for no obvious reason.

**ProSport (-7)** — all bifocals/readers
- Verbena Sunglass Readers (45 missing variation SKUs)
- Bypass Bifocal (36), Gullwing Bifocal (44), Brooklyn Reader (18), Iris Bifocal (12)
- Cooper Polarized Bifocal (8), Sharkbait Polarized Bifocal (7)

**Monster (-5)** — 3 bifocals/readers, 2 regular sunglasses
- Verbena Sun Reader, Cooper Polarized Bifocal, Sharkbait Polarized Bifocal
- Mysto Sunglasses, Nautica Polarized ← not readers, just not synced

**BikerShades (-3)** — all bifocals/readers
- Cabot Photochromic Computer Reader, Edison Photochromic Computer Reader, Hawkeye HD Bifocal
