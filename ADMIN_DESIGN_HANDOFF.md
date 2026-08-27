# Admin Dashboard — Design Handoff (July 3, 2026)

Static HTML mockups only. No JavaScript, no components, no framework. Just HTML and plain CSS. The goal is to nail the layout, hierarchy, and data representation before any code is written.

---

## Design System

### Typography
- **Page titles:** Large serif (e.g. Georgia, or a serif like Playfair Display). Bold weight. Very large — ~48–56px. These are editorial, not functional.
- **Section labels / breadcrumbs:** Small-caps monospace (e.g. `font-family: monospace; font-variant: small-caps; letter-spacing: 0.1em`). Always uppercase. Used for: page category labels above the title ("CATALOGUE HEALTH", "INVENTORY", "TAXONOMY"), column headers in tables.
- **Body / metadata:** Monospace for slugs, SKUs, IDs, dates, counts. Regular sans-serif for names and descriptions.
- **Navigation:** Clean sans-serif, normal weight for inactive, bold for active.

### Color
Almost entirely black and white. Color is used only for semantic meaning — nothing decorative.

| Usage | Color |
|-------|-------|
| Page background | White `#ffffff` |
| Default text | Near-black `#111111` |
| Muted text (slugs, labels, sub-info) | Grey `#888888` |
| Sidebar | White, separated from content by a single vertical rule `#e0e0e0` |
| Dividers / table borders | Light grey `#e0e0e0` thin `1px` lines |
| In stock dot | Green `#16a34a` |
| Out of stock dot | Red `#dc2626` |
| Sale badge | Red outline — border `1px solid #dc2626`, text `#dc2626`, no fill |
| Category pills | Black outline — border `1px solid #111111`, text `#111111`, no fill |
| Active nav item | Bold text + `►` prefix |
| Buttons (primary) | Black background, white text |

### Spacing and layout
- Lots of whitespace — this is editorial, not dense
- Sections separated by thin `1px` horizontal rules, not box shadows or cards
- Stat grids use cell borders (like a table without background color), not individual cards with shadows

### Interaction (static mockup — just show the default/populated state)
- No hover effects needed in static mockup
- Show one expanded order row to illustrate the pattern

---

## Global Layout

Every page shares this shell.

```
┌──────────────────────────────────────────────────────────┐
│ SIDEBAR (~220px, white, right border 1px #e0e0e0)        │
│                                                          │
│  Brand name          ← large, serif or bold sans         │
│  CATALOGUE · ADMIN   ← small caps monospace, grey        │
│                                                          │
│  ────────────────    ← thin rule                         │
│                                                          │
│  [Brand switcher]    ← list of brand names, active bold  │
│  ► BikerShades                                           │
│    Brand Two                                             │
│    Brand Three                                           │
│                                                          │
│  ────────────────                                        │
│                                                          │
│  Navigation                                              │
│    Overview                                              │
│  ► Orders            ← ► prefix + bold = active          │
│    Products                                              │
│    Categories                                            │
│    Analytics                                             │
│                                                          │
│  [bottom]                                                │
│  Admin name          ← e.g. "R. Vance"                   │
│  admin               ← role label, grey                  │
│  Sign out            ← small link                        │
│                                                          │
│ MAIN CONTENT (fills remaining width, padding 48px)       │
│                                                          │
│  SECTION LABEL       ← small caps monospace              │
│  Page Title          ← large serif                       │
│  subtitle / count    ← monospace, grey                   │
│                                                          │
│  [page content]                                          │
└──────────────────────────────────────────────────────────┘
```

---

## Page 1: Orders

**URL:** `/admin/[brandSlug]/orders`

**Section label:** `ORDER MANAGEMENT`
**Title:** `Orders`
**Subtitle:** e.g. `42 orders · $8,430.00 total revenue` in monospace grey

### Status filter tabs

Inline below the subtitle, before the table:
```
All (42)  ·  Processing (12)  ·  Shipped (8)  ·  Delivered (18)  ·  Refunded (3)  ·  Partially Refunded (1)
```
Active tab: bold, underlined. Inactive: grey. Separated by `·`.

### Table

Thin top border rule, then column headers in small caps monospace, then rows separated by `1px` bottom borders.

| Column | Notes |
|--------|-------|
| ORDER | First 8 chars of UUID in monospace. Below it: date in monospace grey, e.g. `Jun 18 · 2:35 PM` |
| CUSTOMER | `shipping_address.name` |
| ITEMS | `2 items` grey monospace |
| TOTAL | `$77.74` right-aligned |
| REFUNDED | `$20.00` in grey if set, `—` if null |
| STATUS | Colored dot + text (see below) |

**Status display:**

| Status | Dot | Text |
|--------|-----|------|
| processing | Blue dot | `Processing` |
| shipped | Purple dot | `Shipped` |
| delivered | Green dot | `Delivered` |
| refunded | Grey dot | `Refunded` |
| partially_refunded | Amber dot | `Partial refund` |

### Expanded order row

Show one row expanded in the mockup to illustrate the pattern. Expanded inline below the row (not a modal), indented, with a light grey left border.

Contents:
- **Shipping address** — stacked: name, line1, line2 (if set), city/state/postal, country
- **Stripe ID** — monospace grey, small: `pi_3abc...`
- **Line items** — small table:
  - Thumbnail (40px square)
  - Name + attribute below in grey (e.g. "Gloss Black / Standard")
  - SKU in monospace grey
  - Unit price, quantity, line total

### Empty state
Centered in the table area. No icon needed — just:
```
No orders yet.
Orders will appear here once customers check out.
```
Both in grey monospace.

---

## Page 2: Products

**URL:** `/admin/[brandSlug]/products`

**Section label:** `INVENTORY`
**Title:** `Products`
**Subtitle:** `589 total · 24 shown` in monospace grey

### Filters row (below subtitle)

```
[Search products or SKU...    ]   [All categories ▾]   [All stock ▾]     24 results
```
Search: plain bordered input. Dropdowns: bordered select. Results count: right-aligned grey monospace.

### Table

| Column | Notes |
|--------|-------|
| (thumbnail) | 48px square image |
| PRODUCT | Name bold. Below: `brand · SKU` in grey monospace |
| CATEGORIES | Outlined pills (black border, no fill), e.g. `Motorcycle` `Polarized` |
| PRICE | If on sale: strikethrough original + red sale price + `SALE` outlined badge. Otherwise: just price. |
| STOCK | Green dot `In stock` or Red dot `Out of stock` |

---

## Page 3: Categories

**URL:** `/admin/[brandSlug]/categories`

**Section label:** `TAXONOMY`
**Title:** `Categories`
**Subtitle:** `16 categories` in monospace grey

### Content

Simple list, not a table. Each row: category name (serif or bold sans, left) + product count (grey monospace, right). Separated by `1px` bottom borders.

For children, indent 24px and use slightly smaller/lighter text.

Example:
```
Sunglasses                                          63 products
    Sport                                           11 products
        Polarized                                    4 products
    Fashion                                         18 products
Accessories                                         23 products
```

No view count needed here for now — keep it simple: name + product count.

---

## Page 4: Analytics

**URL:** `/admin/[brandSlug]/analytics`

**Section label:** `CATALOGUE HEALTH`
**Title:** `Analytics`
**Subtitle:** date or "everything at a glance" in monospace grey

### Stat grid

6 cells in a 3×2 grid, separated by `1px` borders (like a borderless table — the borders are the grid lines, no background fill, no shadows).

| Cell | Label | Value |
|------|-------|-------|
| 1 | ORDERS | Total order count |
| 2 | REVENUE | Total revenue e.g. `$8,430.00` |
| 3 | REFUNDED | Total refunded e.g. `$240.00` |
| 4 | PRODUCTS | Product count |
| 5 | IN STOCK (green dot) | Products with variations |
| 6 | ON SALE (red dot) | Products where sale = true |

Label: small caps monospace, grey, above the number. Number: very large serif (~48px), black.

### Summary sentence

Below the grid, a short paragraph with key numbers highlighted in color:
```
Of 589 products, 542 are in stock and 496 carry an active sale price.
```
Colored numbers match their dot color (green for stock, red for sale).

### Recently Added

Full-width section, "Recently Added" as a medium serif heading with "All products →" link right-aligned.

Thin rule below the heading, then product rows:
- Thumbnail (48px)
- Name bold, below: `brand · SKU` grey monospace
- Right side: strikethrough price + sale price + `SALE` badge (if on sale), then `● In stock` or `● Out of stock`

---

## States to Design

For each page, design the **populated state** with realistic fake data. Show enough rows (6–8) to demonstrate the pattern. No need for separate empty/loading states in the first pass — populated is sufficient.

---

## What's NOT in scope yet

- Any add/edit/delete buttons or forms (categories and products are read-only for now)
- Any "Flagged" page
- Pagination controls (note them as `← 1 2 3 →` placeholder if needed)
- Dark mode
