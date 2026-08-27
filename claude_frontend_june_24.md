# Frontend Build Notes (June 24, 2026)

## Stack

- **Next.js 16 App Router** — server components, Suspense streaming, server actions
- **Supabase** — auth (JWT sessions), storage (images), RLS-protected tables
- **Stripe** — checkout sessions, webhook for order fulfillment
- **Tailwind CSS**

---

## Architecture

### Route Groups

| Group | Layout | Pages |
|-------|--------|-------|
| `(shop)` | Navbar + footer | `/`, `/category/[...path]`, `/product/[slug]`, `/sale` |
| `(bare)` | Logo + back link only | `/checkout`, `/account`, `/signin`, `/order/success`, `/order/failure` |

### Data Fetching Pattern

Server components fetch directly — no client-side data fetching for catalog data. Suspense boundaries stream sections independently.

```
Page (server) → apiFetch → ApiResponse<T> → unwrap .data → render
                                           → !success → notFound() or null
```

Client components (`CartProvider`, `BookmarkProvider`) fetch on mount via server actions.

---

## API Layer (`src/lib/api.ts`)

All functions are server actions (`"use server"`). Two internal helpers:

### `apiFetch<T>`
Public catalog endpoints — builds GET query string, returns `ApiResponse<T>`.

### `authedFetch<T>`
Authenticated endpoints — injects `Authorization: Bearer` from session, POST with JSON body, returns `ApiResponse<T> | null` (null = no session).

### Special Cases

**`validateCart`** — public endpoint but needs a POST body (items array too long for URL). Returns raw `ValidateCartItem[]` directly (no `ApiResponse` wrapper). Status code signals outcome but data is always the full array.

**`createCheckoutSession`** — custom fetch (not `authedFetch`) because the response shape differs by status:
- `200` → `{ success: true, data: { url: "https://checkout.stripe.com/..." } }` (ApiResponse-wrapped)
- `404/409/422` → raw `ValidateCartItem[]` (no wrapper)

Returns `CheckoutResponse | null`.

---

## Type System (`src/lib/types.ts`)

### Catalog

```ts
ListVariation   // used in ProductListItem.variations[] — flat, color swatches only
                // slug is the unique identifier (not id)
                // { option, slug, value?, imageSrc, imageName }

ProductListItem // category/search grid items — one image, optional color variations
ProductDetail   // full item page — structured attributes, all variation images
Variation       // ProductDetail.variations[] — attribute slugs only, no display labels
                // look up labels via product.attributes[].options[].slug
```

### Cart & Bookmarks

```ts
CartAttribute   // { name, option, slug }
                // option = display string ("Gloss Black")
                // slug = URL param ("gloss-black")

CartItem        // productId required (NOT NULL FK in DB)
BookmarkedItem  // productId required (NOT NULL FK in DB)
```

### Orders

```ts
OrderItem       // attribute is string | null ("Gloss Black / Standard")
                // intentionally flat — display only, no deep-link to variation
```

### API Shapes

```ts
ApiResponse<T>     // { success: true; data: T } | { success: false; error: string }
                   // used by all public catalog endpoints and authed endpoints

ValidateCartItem   // { productSlug, sku, exists, priceCents, priceChanged }

CheckoutResponse   // { success: true; url: string }
                 | { success: false; items: ValidateCartItem[] }
```

---

## Key Design Decisions

### Slugs as Variation Identifiers

`ListVariation` uses `slug` as its unique key (not `id`). Slugs are human-readable and stable in URLs:
- Color swatch `key={v.slug}`
- Card href includes `?color=v.slug` when hovering a swatch
- `CartAttribute.slug` stored at add-to-cart time for bag → product deep links

### Selections Stored as Slugs

`ProductDetail` selection state uses slugs throughout:
- URL params: `?color=gloss-black&size=standard`
- Internal state: `{ color: "gloss-black", size: "standard" }`
- Variation matching: `variation.attribute[].slug === selections[attrName]`
- Display labels looked up at render time: `product.attributes.find(a => a.name === n).options.find(o => o.slug === slug).option`

### POST for Authenticated Reads

`getCart`, `getBookmarks`, `getOrders` use POST (not GET) because they need a request body (`{ brandSlug }`). No caching on authenticated data.

### Cart Keyed by `productSlug:sku`

Both localStorage and the merge map use this composite key. SKU identifies the exact variation; productSlug scopes it to the product.

### validate-cart Called Upfront

Called on checkout page mount — marks invalid/changed items before the user hits "Proceed". Checkout itself also validates server-side, returning the same items array on failure so the page can update `invalidItems` and show which items need revision without a generic error.

---

## Navigation Flow

```
Category page
  ProductCard
    → hover color swatch → image preview changes
    → click swatch → router.push(/product/slug?color=gloss-black&path=...)
    → click card → Link href includes hovered color slug + path

Product detail (/product/[slug]?color=gloss-black&size=standard&path=...)
  → URL params are slugs → initialSelections fed into state
  → Add to bag → CartAttribute { name, option, slug } stored
  → Bookmark → { productId, productSlug, name, imageSrc } stored

Bag panel (HeaderIcons)
  → CartAttribute.slug used to build ?color=gloss-black&size=standard link back to product

Checkout page
  → validate-cart on mount → marks invalid items
  → Proceed → createCheckoutSession
      200 → redirect to res.url (Stripe)
      404/409/422 → update invalidItems from res.items array

Account page
  → Orders displayed with attribute string ("Gloss Black / Standard")
  → No links out — intentional
```

---

## Local Storage

Keys: `cart:{brandSlug}`, `bookmarks:{brandSlug}`

Shapes mirror `CartItem[]` and `BookmarkedItem[]`. On sync:
1. Read localStorage
2. Fetch DB (POST with brandSlug)
3. Merge: DB wins on conflict (keyed by `productSlug:sku` for cart, `productSlug` for bookmarks)
4. Write merged result back to localStorage
5. Debounced PUT to DB (800ms) on any change

---

## DB Notes (`src/lib/db/`)

`total_sales` exists on both `products` (simple products) and `variations` (variable products). The `update_total_sales` trigger on `order_items` inserts routes to the correct table by first trying to match a variation by `sku + product_slug + brand_slug`, falling back to the product row.

Query for anything with sales:
```sql
SELECT p.slug, NULL AS sku, p.total_sales
FROM products p WHERE coalesce(p.total_sales, 0) > 0
UNION ALL
SELECT p.slug, v.sku, v.total_sales
FROM variations v JOIN products p ON p.id = v.product_id
WHERE v.total_sales > 0
ORDER BY total_sales DESC;
```
