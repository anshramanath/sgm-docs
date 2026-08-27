# Project Notes — Sunglass Monster Server + proSPORT Frontend (June 14, 2026)

## Overview

A multi-brand sunglasses platform. The server is a Next.js API (deployed on Vercel) that serves product data from Supabase. The frontend (`prosport-frontend/`) is a separate Next.js 16 app that consumes the public API.

---

## Backend — API Endpoints

Base URL: `https://sunglass-monster-server.vercel.app`

All responses follow this shape:
```json
{ "success": true, "data": ... }
{ "success": false, "error": "Message!" }
```

### GET /api/public/categories
Returns the category tree for a brand.

**Params:** `brandSlug` (required)

**Response:**
```json
[
  {
    "id": "uuid",
    "name": "Sunglasses",
    "slug": "sunglasses",
    "sortOrder": 1,
    "children": [
      { "id": "uuid", "name": "Sport", "slug": "sport", "sortOrder": 1 }
    ]
  }
]
```

---

### GET /api/public/products
Returns paginated in-stock products for a brand + category.

**Params:** `brandSlug` (required), `categoryId` (required), `page` (default 1), `size` (default 24, max 100)

**Response:**
```json
{
  "products": [
    {
      "id": "uuid",
      "name": "Airspeed",
      "slug": "airspeed",
      "minPriceCents": 1799,
      "maxPriceCents": 1799,
      "salePriceCents": null,
      "attributes": [{ "name": "color", "options": ["Black HD", "Gunmetal HD"] }],
      "featured": false,
      "sale": false,
      "images": [{ "src": "url", "name": "alt text" }]
    }
  ],
  "page": 1,
  "size": 24,
  "totalPages": 3,
  "totalProducts": 60,
  "hasNextPage": true,
  "hasPreviousPage": false
}
```

---

### GET /api/public/item
Returns full product detail including variations and images.

**Params:** `slug` (required), `brandSlug` (required)

**Response:**
```json
{
  "id": "uuid",
  "name": "Airspeed",
  "sku": null,
  "description": "...",
  "summary": ["bullet 1", "bullet 2"],
  "attributes": [...],
  "featured": false,
  "sale": false,
  "minPriceCents": 1799,
  "maxPriceCents": 1799,
  "salePriceCents": null,
  "stock": null,
  "variations": [
    {
      "id": "uuid",
      "sku": "SKU-123",
      "attribute": [{ "name": "color", "option": "Black HD" }],
      "sale": false,
      "regularPriceCents": 1799,
      "salePriceCents": null,
      "stock": 9,
      "images": []
    }
  ],
  "productImages": [{ "src": "url", "name": "alt text", "sortOrder": 1 }],
  "descriptionImages": [{ "src": "url", "name": "alt text" }]
}
```

---

## Backend — Key Concepts

### Supabase joins
- `table!inner(columns)` — joins and filters: only rows with a matching related row are returned
- `table(columns)` — left join: rows returned even if no related rows exist
- Nested selects like `product_description_images(description_images(src, name))` unwrap join tables
- Use `flatMap` not `map` when unwrapping join tables to avoid `[null]` in results

### Response helpers (`src/lib/api.ts`)
```ts
export function ok(data: unknown, status: number) { ... }
export function err(message: string, status: number) { ... }
```
Always explicit status codes. Error messages end with `!`.

### GET vs POST
- All read endpoints use GET + query params (not POST + body)
- GET requests are cacheable by CDN/browser; POST requests are not
- Query params are for simple scalar values; body is for complex/sensitive data

### HTTP status codes
- `200` — success
- `400` — bad input (missing required param)
- `404` — not found
- `500` — server error

---

## Frontend — Architecture

Located in `prosport-frontend/` (gitignored, local only).

**Stack:** Next.js 16, Tailwind v4, shadcn/ui (via @base-ui/react)

### Key files
```
src/lib/api.ts          — typed fetch functions for all 3 endpoints
src/lib/types.ts        — TypeScript types for all API responses
src/lib/utils.ts        — cn() helper (clsx + tailwind-merge)
src/app/layout.tsx      — root layout: Navbar + main + Footer
src/app/page.tsx        — homepage: products from first leaf category
src/app/category/[...slugPath]/page.tsx  — category page with product grid
src/app/product/[slug]/page.tsx          — product detail page
```

### Components
```
layout/Navbar.tsx        — server component, fetches categories, renders NavMenu
layout/NavMenu.tsx       — client component, shadcn NavigationMenu with recursive dropdowns
layout/Footer.tsx        — static footer
product/ProductCard.tsx  — image, name, price, sale badge, links to /product/[slug]
product/ProductGrid.tsx  — responsive grid of ProductCards + EmptyState
product/ProductDetail.tsx — client component, manages variation selection state
product/ImageGallery.tsx  — client component, thumbnail selector
shared/LoadingSkeleton.tsx — 8-card pulse skeleton for Suspense fallbacks
shared/EmptyState.tsx      — "no products found" message
```

### shadcn UI components used
- `card` — ProductCard, homepage grid
- `separator` — ProductDetail
- `tabs` — ProductDetail description/media tabs
- `skeleton` — LoadingSkeleton
- `navigation-menu` — NavMenu dropdowns

---

## Frontend — Key Concepts

### Next.js 16 patterns
- `params` is a `Promise` — always `await params` before destructuring
- Server components can be `async` and fetch data directly
- Client components need `"use client"` at the top
- Suspense boundaries: wrap async server components to stream content
- `{ next: { revalidate: 60 } }` on fetch — ISR-style caching, stale after 60s

### Server vs client components
- Server: fetches data, no interactivity (Navbar, category page, product page shell)
- Client: needs state or browser APIs (NavMenu, ImageGallery, ProductDetail)
- Pass data from server → client as props to keep data fetching server-side

### Slug-based routing
- Category URLs: `/category/sunglasses/aviator` → `slugPath = ["sunglasses", "aviator"]`
- `[...slugPath]` is a catch-all route that collects any number of URL segments into an array
- `walkTree(nodes, slugPath)` walks the category tree to resolve slugs → CategoryNode
- Product URLs: `/product/[slug]` — slug is unique per brand

### Variation selection logic
- Each variation has `attribute: [{ name, option }]`
- Extract unique attribute names across all variations
- For each attribute, show all options as buttons
- Available options = options that appear in at least one variation compatible with current other selections
- A variation is resolved only when all attributes have a selection → price and images update

### Tailwind v4 CSS variables
```css
@import "tailwindcss";

@theme inline {
  --color-background: var(--background);
  --color-muted: var(--muted);
  /* maps CSS vars to Tailwind utilities like bg-muted, text-foreground */
}

:root {
  --background: oklch(1 0 0);
  /* define actual values here */
}
```
Without `@theme inline`, utility classes like `bg-muted` don't work in v4.

### cn() utility
```ts
cn("p-4 p-2")        // → "p-2"  (tailwind-merge resolves conflicts)
cn("text-sm", isLarge && "text-lg")  // → "text-sm text-lg" or "text-sm"
```

---

## Deployment

- Backend on Vercel, auto-deploys from `main` branch
- `prosport-frontend/` is gitignored — lives locally only
- Root `tsconfig.json` excludes `prosport-frontend` to prevent it being picked up by the server build
- Supabase storage hostname whitelisted in `next.config.ts` for `next/image`
