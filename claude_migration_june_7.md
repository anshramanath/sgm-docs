# Migration Learnings (June 7, 2026)

## Project Overview

Migrating WooCommerce product catalogues for three brands (proSPORT, Sunglass Monster, BikerShades) into a Supabase-backed multi-brand platform. Pipeline lives in `final-migration/`. Brand is set via the `BRAND` env var.

---

## Pipeline Stages

```
fetch-products → create-items → validate-items → remove-items → merge-attributes → dedup-images → generate-content → (import)
```

Each stage reads from the previous stage's output directory under `data/{brand}/`.

---

## Data Learnings

### Products

- **Variable products** have `sku: null` at the product level — SKU and pricing live on variations only.
- **Simple products** have SKU, price, and a non-empty variations array at the product level.
- Checks (Veeqo presence, SKU structure) apply at variation level for variable products, product level for simple products.
- Bad variations (not in Veeqo, not an object) are dropped silently. A variable product needs at least 2 surviving variations to proceed.

### Pricing

- `minPriceCents` and `maxPriceCents` are derived from surviving child variations for variable products.
- Null price is intentional — flags products that need attention before import.
- Morocco Mirror has a $0.00 price on proSPORT and Monster — needs fixing before import.

### Images

- Product images deduped by `src` URL (keep first).
- Variation images that match any parent `productImages` src are removed entirely — empty `variationImages` means inherit from parent.
- ~7,525 variation image duplicates removed across all brands.

### Attributes

- WooCommerce attribute names were inconsistent across brands. Merged to canonical names:
  - `color` — 5 variants consolidated
  - `quantity` — 4 variants consolidated (+ option rewrite for "Qty (packs of 3)": "1 pack (3 masks total)" → "3 masks" etc.)
  - `power`, `transition`, `size` — renamed
- After merge, zero conflicts within a single product (deduplication handled in `merge_product_attrs()`).

### Removed Items

Tracked in `docs/remove_items.md`. Key removals:
- Shipping label variations (LABEL-FEDEX2DAYS, LABEL-UPSOVERNIGHT)
- VShield slugs on Monster + BikerShades (conflicting qty attributes)
- 7-eye-replacement-foam (discontinued)
- Wiley X Saint for RX + 29 Rx simple SKUs on BikerShades (prescription products, not suitable for general sale)

---

## Content Generation (MCP Server)

### Architecture

- `mcp/server.py` is a FastMCP server Claude Desktop spawns as a local process.
- Claude Desktop communicates with it over **stdio** (stdin/stdout JSON) — no HTTP, no network from Claude Desktop's side.
- The Python process has full local network access — it fetches product images and passes raw bytes to Claude Desktop via FastMCP's `Image()` class.
- Claude Desktop forwards image bytes to Anthropic's servers with the prompt. The model sees the image and generates content.
- All file reads and writes happen in the Python process — Claude Desktop never touches the filesystem directly.

### Why MCP over a script

- **Images**: Claude can see product photos, which directly informs better descriptions (frame shape, lens color, foam seal, style).
- **Scale**: 882 products across 3 brands. Looping in Claude Desktop doesn't fill up a single context window.
- **Autonomy**: Claude Desktop runs the loop unattended — get_next_product → generate → save_content → repeat.

### Tools

- `get_next_product(brand)` — returns the next item needing content + product image. Returns `list` (text + Image).
- `save_content(brand, slug, description, summary)` — validates then writes to `generate-content/items.json`.
- `get_progress(brand)` — returns `done/total, remaining` as a plain string.

### Validation in save_content

Rejects saves that fail:
- Description over 80 words
- Description contains measurements (Lens Width, mm, etc.)
- Description uses banned phrases (available in, choose from, comes in)
- Fewer than 4 or more than 6 bullets
- Bullet ends with a period
- Bullet starts lowercase
- Bullet over 12 words
- Bullet uses banned phrases

### Prompt Rules That Matter

- **No attributes in the prompt** — passing attribute options caused Claude to generate "Available in black, silver, and tortoise" bullets.
- **Bullets must describe features true of ALL variants** — not choices or options.
- **Never use "Available in", "Choose from", "Comes in"** — banned in both bullets and description.
- **Do not infer coatings or certifications** — Claude called clear computer readers "blue-light blocking" from the image alone. Added explicit key concept: "Computer readers have plain clear lenses — do not call them blue-light blocking."
- **Only state what you can see or what the product name states** — prevents hallucinated specs.

### FastMCP Notes

- Return type annotation on tools matters — `-> str` with a list return caused a validation error. Changed to `-> list`.
- `Image(data=raw_bytes, format="jpeg")` — FastMCP expects raw bytes (not base64 string) and short format string (not full MIME type).
- `mcp.run()` defaults to stdio transport. SSE (HTTP) transport: `mcp.run(transport="sse", host="127.0.0.1", port=8000)`.
- Claude Desktop config: `~/Library/Application Support/Claude/claude_desktop_config.json`. Claude Desktop overwrites this on launch — always Cmd+Q before editing.

### wasMissing Flags

- `wasMissingDescription` and `wasMissingSummary` set once at init in `generate_content.py`, never changed.
- Used post-generation to compare what was originally missing vs. what was generated.
- `generate_content.py` always nulls `description` and `summary` on init — everything is regenerated from scratch regardless of what WooCommerce had.

---

## WooCommerce Data Quality Issues

- Many product descriptions were raw dimension data: "Lens Width: 53 mm Bridge Width: 22 mm..."
- Some descriptions were generic AI-generated WooCommerce copy ("Step into the spotlight with our chic sunglasses").
- Placeholder text leaked through: "product short description" as a summary bullet.
- Old WooCommerce descriptions sometimes ended up as summary bullets when the data was partially structured.
- Permalink field was not populated in the fetched data — use slug to construct product URLs manually.

---

## Pending Before Import

- Finish content generation for all three brands (prosport in progress, monster and bikershades not started)
- Categorize products (categories field still empty array for all items)
- Download images to local storage
- Resolve duplicate slugs: Monster has 'Vato' collision; BikerShades has 3 duplicate slugs
- Fix Morocco Mirror $0.00 price (flagged in prosport and monster)
- Fix missing product images: Rocket High Wrap + Leopard Print Case (monster), Angel Rhinestones COMBOS (bikershades)
- Import to Supabase (field mapping: camelCase pipeline → snake_case DB columns)
