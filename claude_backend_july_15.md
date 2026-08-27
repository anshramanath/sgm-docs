# Dev Log — Product Admin & Server Action Patterns (July 15, 2026)

## What Was Built

### Product detail page
Full create/edit flow at `/admin/[brandSlug]/products/[productId]`, with `"new"` as a special ID for creation.

**Files:**
- `src/lib/admin/product-detail.ts` — server actions: `getProductDetail`, `saveProduct`, `deleteProduct`, `uploadImage`
- `src/app/admin/[brandSlug]/products/[productId]/page.tsx` — server component, fetches product + categories in parallel
- `src/app/admin/[brandSlug]/products/[productId]/product-form.tsx` — full client form

**Form handles:**
- Simple vs variable products (no variations = simple; variations = variable)
- Product images and per-variation images, uploaded to Supabase storage before save
- Attributes with optional color-picker value when `name === "color"`
- Categories (multi-select tags)
- Sale toggle with sale price field

### Products list updates
- Clicking a row navigates to the product detail page
- "New product" button navigates to `/products/new`

### Orders table cleanup
Removed inline wrapper functions (`save`, `undo`) from `orders/page.tsx` and prop-threading through `OrdersTable`. The client component now imports `saveFulfillment` and `undoFulfillment` directly.

### Products list cleanup
Removed `brandSlug` and `loadMore` props from `ProductsList`. The component now reads `brandSlug` from `useParams()` and calls `getProducts` directly.

---

## Patterns & Lessons

### `"use server"` modules — what you can and can't export

All exports from a `"use server"` file are treated as server actions at runtime. **You cannot re-export types** from them — Next.js will try to resolve them as functions and throw:

```
Export ProductDetailData doesn't exist in target module
```

Put shared types in a plain `types.ts` file and import from there in both the server action module and client components.

### Client components can import server actions directly

No need to pass server actions as props through server components. Client components can import from `"use server"` modules directly:

```ts
// ✅ client component
import { saveFulfillment } from "@/lib/admin/orders";
```

Prop-threading server actions is only necessary when the action needs to close over a value from the server component (like a route param). Even then, consider making the param explicit and importing directly instead.

### Inline `"use server"` functions close over parent scope

```ts
// page.tsx (server component)
async function loadMore(search, categoryId, offset) {
  "use server";
  return getProducts(brandSlug, { ... }); // closes over brandSlug
}
```

This works but means the function lives in the server component and must be passed as a prop. The cleaner alternative: export a named server action that accepts `brandSlug` explicitly, and import it in the client.

### `useParams()` vs prop from server

Both are O(1) — one reads from React context, the other is prop drilling. No performance difference. `useParams()` couples the component to the URL structure; a prop makes it more portable. For components that are inherently tied to a route (like an admin panel scoped to a brand), `useParams()` is the cleaner choice.

### Supabase storage — per-brand buckets

Storage is organized by brand slug, not a single `products` bucket:

```
bikershades/
sunglass-monster/
prosport-sunglasses/
```

`uploadImage` reads `bucket` from `FormData`. The client appends `fd.append("bucket", brandSlug)` before calling the server action. Image paths within a bucket:

```
{productId}/{timestamp}-{filename}           // product images
{productId}/variations/{variationId}/{...}    // variation images
```

### Pre-generating product UUID client-side

For new products, the UUID is generated in the client with `crypto.randomUUID()` before the product is saved. This allows image uploads to go to a deterministic path before the product record exists in the database.

### Simple vs variable save logic

- **Simple**: `variations.length === 0`. SKU lives on the product row. All existing variations are deleted on save.
- **Variable**: SKU is `null` on the product row. SKUs and prices live on each variation. `attributes` jsonb is derived from variation data on save.
- Existing variation IDs are preserved on update to keep `total_sales`. Only variations whose `id` starts with `"new-"` are inserted.
- Variation images are re-associated after save by doing a fresh `SELECT id, sku` and matching by SKU.
