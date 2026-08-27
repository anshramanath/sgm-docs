# Sunglass Server — Dev Log (July 20, 2026)

## Product Admin Page (`/admin/[brandSlug]/products/[productId]`)

### Architecture

- **Server component** (`page.tsx`) fetches brand, categories, and product data, then renders `<ProductForm>`.
- **Client component** (`product-form.tsx`) holds all form state and calls server actions directly — no prop threading needed.
- **Server actions** (`lib/admin/product-detail.ts`) handle all DB reads/writes. Marked `"use server"` — cannot re-export types from this module (causes runtime error); all shared types live in `lib/types.ts`.

---

### Product Model: Simple vs Variable

- `isSimple = variations.length === 0`
- Simple: has `sku`, `regular_price_cents`, `sale_price_cents` at the product level.
- Variable: `sku` and `sale_price_cents` are always `null` at the product level. Price range (`min_price_cents`, `max_price_cents`) is derived from variations.
- `sale = true` at the product level means at least one variation is on sale.
- `total_sales` is never written by admin saves — only incremented by the order flow. New variation inserts use `total_sales: 0`.

---

### Price Handling

- All prices stored as cents (integers) in the DB.
- `centsToDollars(cents)` → `(cents / 100).toFixed(2)` — always 2 decimal places.
- `dollarsToCents(str)` → `Math.round(parseFloat(str) * 100)`, returns `0` for invalid/empty input.
- `formatPrice(val)` → applies `parseFloat(val).toFixed(2)` on blur so inputs always display 2 decimal places when leaving the field.
- `dollarsToCents("") = 0`, so `|| null` is used when saving optional prices to avoid storing `0` instead of `null`.
- Sale price is always visible but disabled when not on sale — preserves the entered value so the user doesn't need to re-enter it when toggling sale back on.
- Variable product effective price per variation: `sale && salePriceCents ? salePriceCents : regularPriceCents`.

---

### Image Uploads

- Images upload immediately on file selection via `uploadImage` server action using `FormData` (required for `File`/`Blob` — JSON can't serialize binary).
- Storage bucket = `brandSlug` (e.g. `bikershades`, `sunglass-monster`). Named this way so the bucket is always the brand's slug — a deliberate early architectural decision.
- Path: `{productId}/{timestamp}-{safeName}` for product images, `{productId}/variations/{variationId}/{timestamp}-{safeName}` for variation images.
- `upsert: true` on upload — timestamp-scoped paths make collisions effectively impossible in practice.
- Removed images are not deleted from storage immediately — if the user removes then doesn't save, the storage URL would be invalidated. A periodic cleanup job can remove unlinked images.
- Product UUID is pre-generated client-side (`crypto.randomUUID()`) so images can be uploaded before the product row exists in the DB.
- Body size limit raised to `10mb` in `next.config.ts` (`experimental.serverActions.bodySizeLimit`).
- Hidden `<input type="file">` controlled via `useRef` — a custom-styled button calls `.click()` on the ref to open the file picker. `e.target.value = ""` resets after selection so picking the same file twice still fires `onChange`.

---

### Variations

- Each variation holds: `sku`, `regularPrice` (string), `salePrice` (string), `sale` (bool), `attrs` (`FormAttr[]`), `images`.
- New variations are seeded with `id: "new-{uuid}"` client-side. On save, new ones are inserted (with `total_sales: 0`), existing ones are updated by UUID, removed ones are deleted.
- Variation images are replaced entirely on save, looked up by SKU (since new variations don't have a DB UUID yet).
- When going from variable → simple (last variation removed), its data (sku, prices, sale) is inherited back into the product-level fields.
- When going from simple → variable (first variation added), product-level data seeds the first variation.

---

### Attributes

- Five fixed suggestions: `color`, `power`, `transition`, `quantity`, `size`.
- Stored lowercase in DB; displayed capitalized in the UI.
- Color attribute shows an extra color picker. `value` defaults to `#000000` when "color" is selected so the picker and the stored value are always in sync.
- `deriveAttributes`: builds product-level `attributes` jsonb from variation data. Groups options by attribute name, deduplicates by slug. First occurrence wins when two options share a slug.
- Slug used for dedup because two options can have the same display name with different casing.

---

### Validation (in `handleSave`)

- Product name required.
- At least one image required.
- Variable product must have ≥ 2 variations.
- Simple product: SKU required; if on sale, sale price must be > 0 and < regular price.
- Each variation: SKU required; regular price > 0; if on sale, sale price > 0 and < regular price; each attribute row must have both name and option (and hex value for color); at least one complete attribute required.
- Errors displayed below save/delete buttons.

---

### UI Patterns

- `NavProgress` active while saving or uploading.
- Save disabled while uploading.
- Category dropdown: max height 320px with scroll.
- Attribute name: select-only dropdown (no free typing) with a fixed-position backdrop div to close on outside click. Toggle logic simplified — backdrop always intercepts clicks before the trigger, so `onClick` just sets the key (never needs to toggle to null).
- Price inputs: `step="1"`, `onBlur` reformats to 2 decimal places.
- `useParams()` preferred over prop-passing for route params in client components tightly coupled to the route.

---

### Type Organization

- Shared types (used across ≥ 2 files): `types.ts`.
- File-local types: live in their own file.
- `"use server"` modules cannot re-export types — runtime error.
- `FormImage`, `FormAttr`, `FormVariation`: form-only types, belong in `product-form.tsx`.
- `LeafEntry`: only used alongside `collectLeaves` in `utils.ts`.
- `ProductDetailVariation`: unused, removed.

---

### Analytics

- `getTopCategoryViews`: fetches all categories in one query, builds a `catMap` keyed by ID, walks parent chain to build full path (e.g. `Sunglasses / Sport / Polarized`). Needed because two leaf categories can share the same name and only their path makes them unique.
- Sort and slice in JS after fetching all — can't filter to leaves at the query level without losing the parent chain needed to build paths.
- `RankedList` uses index as key — data is static server-rendered with no reordering.
- `key={r.name}` was a bug: product names aren't unique in the top sales list (same product appears once per SKU).

---

### React Patterns

- `key={index}` is safe for static lists with no reordering. Use stable IDs (e.g. `o.id`) for lists that can reorder or have identity (orders, categories).
- `useRef` for DOM nodes that don't need to trigger re-renders (e.g. hidden file input).
- `useState` initial value only runs on first render — subsequent renders ignore it.
- `onBlur` vs `onChange` for formatting: `onChange` with `toFixed(2)` would reformat on every keystroke, making the field untyable.
- Spread (`...a`) creates a shallow copy — safe for immutable state updates on flat objects.
- `Array.prototype.unshift` prepends to front; useful when walking a tree bottom-up (child → root) to build a path in top-down order.
