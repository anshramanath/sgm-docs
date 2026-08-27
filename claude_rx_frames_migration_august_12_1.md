# Prescription Frames (August 12, 2026)

A catalog of prescription-compatible eyewear frames with associated images, sizing, rx ranges, and color options. Used to power the prescription frames storefront.

---

## Overview & Purpose

This dataset was built to seed the prescription frames feature of the sunglass e-commerce platform. The goal is to migrate the data into the production database once the catalog is complete and validated. Until then, `prescription-frames.json` and `images/` serve as the staging layer — a clean, human-audited source of truth before anything hits the DB.

### Data Sourcing Process

All frame data was sourced manually from third-party vendor websites (BikerArmour, 7eye, Wiley X, Ziena) by reviewing individual product pages. The ingestion workflow per frame:

1. **Screenshot** — a product page screenshot is taken showing the frame name, price, rx range, size, and open color dropdown
2. **Image** — the frame's product image is saved as `image.png` and dropped into the `images/` folder
3. **Extraction** — name is cleaned (strip "for Rx", bracket content, etc.), non-sale price is used, rx range and size are read from the bullet points, colors are read from the open dropdown
4. **Rename** — `image.png` is renamed to `<slug>.png` to match the entry
5. **Append** — a new JSON entry is appended to the end of `prescription-frames.json`
6. **Validate** — `python3 validate.py` is run — all checks must pass before moving on

### Why JSON + Images First

Rather than writing directly to the database, the flat-file approach allowed:
- Easy review and correction of individual entries without DB tooling
- A validation script to enforce schema consistency before migration
- A single source of truth that can be inspected, diffed, and corrected in a text editor
- No partial or malformed data reaching the production DB

The JSON and images folder are the migration input — once complete they will be bulk-inserted into the database with images uploaded to storage.

---

## Data File

`prescription-frames.json` — array of frame objects, one per entry.

### Schema

```json
{
  "name": "Brand Frame Name",
  "slug": "brand-frame-name",
  "image": "images/brand-frame-name.png",
  "priceCents": 12900,
  "size": "MED-LG",
  "rxLow": -3.50,
  "rxHigh": 3.50,
  "colors": [
    { "slug": "matte-black", "option": "Matte Black", "value": "#2b2b2b" }
  ]
}
```

### Field Rules

| Field | Rule |
|---|---|
| `name` | Brand + frame name. Strip "for Rx", "for High Rx", "for High Power Rx", and any bracket content `[...]`. Must be non-empty. |
| `slug` | Slugified name: lowercase, non-alphanumeric sequences → `-`, trim leading/trailing `-`. Must be unique. |
| `image` | Always `images/<slug>.png`. File must exist on disk. |
| `priceCents` | Non-sale price in cents (e.g. $129.00 → `12900`). Must be above $35.00. |
| `size` | Dash-separated size tokens in ascending order. Tokens: `XS`, `SM`, `MED`, `LG`, `XL`, `XXL`. Always `MED` not `MD`. |
| `rxLow` | Minimum sphere power (negative). Must be present and less than `rxHigh`. |
| `rxHigh` | Maximum sphere power. Must be present. |
| `colors` | Non-empty array. Each color's `slug` must be the slugified `option`. `value` must be a valid 6-digit hex (`#rrggbb`). |

---

## Naming Rules

- Strip suffixes: `for Rx`, `for High Rx`, `for High Power Rx`
- Strip bracket content: `[SM – MED]`, `(FITS MEDIUM TO LARGE HEADS)`, etc.
- High power variants: append `High Power` to avoid slug collision with base model (e.g. `7eye Briza` → `7eye Briza High Power`)
- No "Rx" anywhere in the name

### High Power Variants

When a frame has both a standard and high-power version, both are added as separate entries with `High Power` appended to the high-power name. Rx ranges differ:

| Frame | rxLow | rxHigh |
|---|---|---|
| Standard | -3.50 to -7.00 typical | 3.50 to 7.00 typical |
| High Power | -5.00 to -8.00 | 5.00 to 8.00 |

Some Wiley X high power frames have asymmetric ranges (e.g. -7.00/+5.00).

---

## Size Convention

Source sites use inconsistent tokens. Normalize on import:

| Source | Stored |
|---|---|
| `MD` | `MED` |
| `XLG` | `XL` |
| `XSM` | `XS` |

Valid tokens in order: `XS < SM < MED < LG < XL < XXL`

---

## Adding a Frame

1. Drop `image.<ext>` into `prescription-frames/images/`
2. Send the product page screenshot
3. Extract: name (cleaned), non-sale price, size, rx range, open color dropdown
4. Rename image: `mv image.<ext> images/<slug>.<ext>`
5. Append entry to end of `prescription-frames.json`
6. Run `python3 validate.py` — all checks must pass

### Flags Before Adding

- **No image file**: notify before proceeding
- **Color dropdown closed**: notify and ask to open it
- **Rx range not stated on page**: flag and do not assume — skip or ask

### Alternate Fit / Duplicate Variants

Only add if meaningfully different from the base model (different size or distinct color set). Skip if it is a pure subset of the base at the same price and size.

---

## Brands

### BikerArmour
- Price range: $39–$70
- Rx range: ±1.50 to ±8.00
- Sizes: XS–XXL
- Colors: mostly Black; some Red, Blue, Pink, Orange, Purple

### 7eye
- Price range: $109–$129
- Rx range: ±3.50 standard; ±5.00–7.00 high power
- Sizes: XS–XL
- Colors: Gloss/Glossy Black, Matte Black, Charcoal, Dark Tortoise, Sunset Tortoise, Gray Tortoise, etc.

### Wiley X
- Price range: $109–$149
- Rx range: ±3.50 standard; asymmetric -7.00/+5.00 high power
- Sizes: XS–XXL
- Colors: Matte Black, Gloss Black, Matte Grey, Gloss Demi, Matte Kryptek, etc.

### Ziena
- Price range: $180
- Rx range: varies per frame — not always stated on product page, must be confirmed before adding
- Sizes: MED-LG
- Colors: Gloss Black, Tortoise, Wood Grain Veneer, Silver Titan, Merlot
- Note: color options on source site include eyecup variants (e.g. "Gloss Black w/ Frost Eyecup") — simplify to frame color only

---

## Validation Script

`validate.py` — run after every addition to catch data issues early.

```bash
python3 prescription-frames/validate.py
```

### Checks

| Check | Detail |
|---|---|
| Name present | Must be non-empty |
| Unique slugs | Tracks seen slugs in a set; errors on duplicate |
| Slug matches name | Slugified name must equal stored slug |
| Image field correct | Must be `images/<slug>.png` |
| Image file exists | Checks disk for the `.png` |
| Price present | `priceCents` field must exist |
| Price above minimum | Must be strictly above $35.00 |
| Size tokens valid | Each token must be in `[XS, SM, MED, LG, XL, XXL]` |
| Size ascending | Tokens must be in order (SM-LG valid, LG-SM not) |
| rxLow present | Field must exist |
| rxHigh present | Field must exist |
| rxLow < rxHigh | Enforced numerically |
| Colors non-empty | Array must have at least one entry |
| Color option present | Each color must have an `option` field |
| Color slug matches option | Slugified `option` must equal `slug` |
| Color value is valid hex | Must match `#[0-9a-f]{6}` |

### Slugify (matches app's `src/lib/utils.ts`)

```python
def slugify(name):
    s = name.lower().strip()
    s = re.sub(r"[^a-z0-9]+", "-", s)
    s = re.sub(r"^-+|-+$", "", s)
    return s
```

---

## Database Migration

The `prescription_frames` table was added in `005_prescription_frames.sql` and mirrored into `initial_schema.sql`. The `drop_schema.sql` was also updated.

### Table Schema

```sql
create table prescription_frames (
  id          uuid        primary key default gen_random_uuid(),
  brand_slug  text        not null references brands(slug) on delete cascade,
  name        text        not null,
  slug        text        not null unique,
  image_src   text        not null,
  price_cents int         not null,
  size        text        not null,
  rx_low      numeric     not null,
  rx_high     numeric     not null,
  colors      jsonb       not null,
  created_at  timestamptz not null default now(),
  updated_at  timestamptz not null default now()
);

alter table prescription_frames enable row level security;
```

RLS is enabled with no policies — all access goes through the admin client which bypasses RLS entirely, so a public read policy would be dead code.

The `colors` JSONB column stores the same array structure as the JSON file: `[{ "slug", "option", "value" }]`.

---

## Color Hex Reference

| Option | Hex |
|---|---|
| Black | `#000000` |
| Gloss Black / Glossy Black | `#000000` |
| White | `#ffffff` |
| Matte Black | `#2b2b2b` |
| Charcoal / Matte Charcoal | `#3d3d3d` |
| Matte Graphite | `#3d3d3d` |
| Matte Grey / Matte Gray | `#888888` |
| Gray Tortoise | `#5a5a5a` |
| Silver Titan | `#9ea0a3` |
| Crystal Metallic | `#a8a8b0` |
| Matte Cool Gray | `#8a9298` |
| Matte Slate | `#6b7c8d` |
| Gloss Demi / Matte Demi | `#8b6914` |
| Tortoise | `#8b6835` |
| Dark Tortoise | `#4a3010` |
| Sunset Tortoise | `#c17a2a` |
| Light Tortoise | `#c89030` |
| Brown Tortoise | `#7a4818` |
| Black Stripe Tortoise | `#2a1a08` |
| Black Pearl / Black Fade | `#1a1a1a` |
| Black Crystal | `#1a1a1a` |
| Kryptek Typhon | `#1a1a1a` |
| Matte Woodgrain / Wood Grain Veneer | `#7b5c3a` |
| Matte Kryptek | `#7a7a5a` |
| Matte Utility Green | `#4a5e3a` |
| Hickory Brown | `#6b3a1f` |
| Merlot | `#8b2635` |
| Red | `#cc0000` |
| Blue | `#1a4aaa` |
| Purple | `#7b2d8b` |
| Orange | `#e87722` |
| Pink | `#e75480` |
