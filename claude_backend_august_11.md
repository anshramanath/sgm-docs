# Prescription Frames Data (August 11, 2026)

78 frames across 4 brands: Wiley X, BikerArmour, 7eye, and Wiley X Saint.

---

## File Structure

```
prescription-frames/
  prescription-frames.json   # all frame data
  images/                    # one image per frame, named <slug>.<ext>
```

---

## JSON Schema

```json
{
  "name": "Brand Frame Name",
  "slug": "brand-frame-name",
  "image": "images/brand-frame-name.png",
  "priceCents": 10900,
  "size": "MED-LG",
  "rxLow": -3.50,
  "rxHigh": 3.50,
  "colors": [
    { "slug": "matte-black", "option": "Matte Black", "value": "#2b2b2b" }
  ]
}
```

| Field | Type | Notes |
|---|---|---|
| `name` | string | Display name, cleaned (see rules below) |
| `slug` | string | Kebab-case, used as identifier and image filename |
| `image` | string | Always `images/<slug>.<ext>` |
| `priceCents` | number | Non-sale (regular) price in cents |
| `size` | string | Head size range (see Size section) |
| `rxLow` | number | Minimum sphere power (negative) |
| `rxHigh` | number | Maximum sphere power (positive) |
| `colors` | array | At least one color object |
| `colors[].slug` | string | Kebab-case color identifier |
| `colors[].option` | string | Display name shown to customer |
| `colors[].value` | string | Hex color for UI swatch |

---

## Name Rules

- Strip `for Rx`, `For Rx`, `for High Rx`, `For High Power Rx` from the product title
- Strip brackets `[...]` and their contents (size ranges)
- Strip parentheses `(...)` and their contents (e.g. `(Alternative Fit)`) — **unless** needed to distinguish from another entry with the same base name
- **High power variants**: if a frame has a standard and high-power version, append `High Power` to the name (e.g. `7eye Diablo High Power`, `7eye Cape High Power`, `BikerArmour Air Boss High Power`)
- Do **not** append "Rx" anywhere in the name

---

## Size Rules

- Taken from the bracket in the product title: `[SM – MED]` → `SM-MED`
- All caps, no spaces around dashes
- Standardized abbreviations — always use `MED`, never `MD`
- Common values: `XS-SM`, `SM-MED`, `MED-LG`, `MED-XL`, `MED-XLG`, `MED-XXL`, `LG-XL`, `LG-XXL`, `LG-XLG`, `XSM-MED`

---

## Price Rules

- Always use the **non-sale (regular/strikethrough) price**
- Ignore sale prices shown in red
- Color add-on costs (e.g. `[+ $20]`) are not reflected — `priceCents` is the base frame price only

---

## Color Rules

- Use the exact display name from the dropdown as `option`
- `slug` is the kebab-case version of `option`
- `value` is a representative hex — not pixel-perfect, just recognizable for UI swatches
- If the dropdown is not open in the screenshot, flag it before proceeding
- If the dropdown has no options (empty), infer from the frame image (typically black)

### Reference Hex Values

| Color Name | Hex |
|---|---|
| Black / Gloss Black / Glossy Black | `#000000` |
| Matte Black | `#2b2b2b` |
| Matte Charcoal / Charcoal | `#3d3d3d` |
| Matte Graphite | `#3d3d3d` |
| Matte Grey / Matte Gray | `#888888` |
| Matte Slate | `#6b7c8d` |
| Matte Cool Gray | `#8a9298` |
| Crystal Metallic | `#a8a8b0` |
| Matte Black (Tomahawk) | `#2b2b2b` |
| Tortoise | `#8b6914` |
| Gloss Demi / Matte Demi | `#8b6914` / `#7a5c1e` |
| Dark Tortoise | `#4a3010` |
| Brown Tortoise | `#7a4818` |
| Light Tortoise | `#c89030` |
| Sunset Tortoise | `#c17a2a` |
| Black Stripe Tortoise | `#2a1a08` |
| Gray Tortoise | `#5a5a5a` |
| Black Pearl | `#1a1a1a` |
| Black Fade | `#1a1a1a` |
| Kryptek Typhon | `#1a1a1a` |
| Matte Kryptek | `#7a7a5a` |
| Hickory Brown | `#6b3a1f` |
| Matte Utility Green | `#4a5e3a` |
| Matte Woodgrain | `#7b5e3c` |
| Red | `#cc0000` |
| Blue | `#1a4aaa` |
| Orange | `#e87722` |
| Pink | `#e75480` |
| Purple | `#7b2d8b` |

---

## Rx Range Notes

- Most frames: `rxLow: -3.50, rxHigh: 3.50`
- High power variants: `rxLow: -6.00` to `-7.00`, `rxHigh: 6.00` to `7.00`
- Some frames have asymmetric ranges (e.g. Wiley X Kingpin: `-7.00 / +5.00`)
- Check the bullet point "Accommodates power up to +/- X.XX" on each product page

---

## Process (Adding a Frame)

1. Drop image named `image.<ext>` into `prescription-frames/images/`
2. Send product page screenshot (dropdown open)
3. Rename `image.<ext>` to `<slug>.<ext>`
4. Append JSON entry to `prescription-frames.json`

**Flags before proceeding:**
- Color dropdown not open → ask user to open it
- No `image.*` file found → notify user

---

## Brands

| Brand | Price Range | Typical Size |
|---|---|---|
| BikerArmour | $39–$69 | SM-MED to LG-XL |
| 7eye | $109–$119 | SM-MED to MED-XLG |
| Wiley X | $109–$149 | XS-SM to MED-XL |
