# Sunglass Server — Dev Log (July 22, 2026)

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
- `formatPrice(val)` → applies `parseFloat(val).toFixed(2)` on blur so inputs always display 2 decimal places when leaving the field. Uses `onBlur` not `onChange` because reformatting on every keystroke makes the field untyable.
- `dollarsToCents("") = 0`, so `|| null` is used when saving optional prices to avoid storing `0` instead of `null`.
- Sale price is always visible but disabled when not on sale — preserves the entered value so the user doesn't need to re-enter it when toggling sale back on.
- Variable product effective price per variation: `sale && salePriceCents ? salePriceCents : regularPriceCents`.

---

### Image Uploads

- Images upload immediately on file selection via `uploadImage` server action using `FormData` (required for `File`/`Blob` — JSON can't serialize binary).
- Storage bucket = `brandSlug` (e.g. `bikershades`, `sunglass-monster`). Named this way so the bucket is always the brand's slug — a deliberate early architectural decision.
- Paths: `{productId}/{timestamp}-{safeName}` for product images, `description/{timestamp}-{safeName}` for description images, `{productId}/variations/{variationId}/{timestamp}-{safeName}` for variation images.
- `upsert: true` on upload — timestamp-scoped paths make collisions effectively impossible in practice.
- Removed images are not deleted from storage immediately — if the user removes then doesn't save, the storage URL would be invalidated. A periodic cleanup job can remove unlinked images.
- Product UUID is pre-generated client-side (`crypto.randomUUID()`) so images can be uploaded before the product row exists in the DB.
- Body size limit raised to `10mb` in `next.config.ts` (`experimental.serverActions.bodySizeLimit`).
- Hidden `<input type="file">` controlled via `useRef` — a custom-styled button calls `.click()` on the ref to open the file picker. `e.target.value = ""` resets after selection so picking the same file twice still fires `onChange`.

---

### Description Images

- Stored in a brand-level shared table `description_images` (unique on `brand_slug, src`) — the same image can be reused across products.
- Linked to products via the `product_description_images` junction table.
- On save: delete all junction rows for the product, then for each description image upsert into `description_images` (idempotent on `brand_slug, src`), then insert the junction row.
- Supabase always returns nested embedded resources as arrays even for many-to-one joins — use `flatMap` not `map` when unwrapping.

---

### Save Logic & Atomicity

- Product images, categories, and description images all use a delete-all + re-insert pattern on save. This is not atomic — if the insert fails after the delete, those rows are lost.
- The real fix is a Postgres RPC function that wraps both operations in a transaction. Not yet implemented.
- Variation images are replaced entirely on save, looked up by SKU (since new variations don't have a DB UUID yet).

---

### Variations

- Each variation holds: `sku`, `regularPrice` (string), `salePrice` (string), `sale` (bool), `attrs` (`FormAttr[]`), `images`.
- New variations are seeded with `id: "new-{uuid}"` client-side. On save, new ones are inserted (with `total_sales: 0`), existing ones are updated by UUID, removed ones are deleted.
- When going from variable → simple (last variation removed), its data (sku, prices, sale) is inherited back into the product-level fields.
- When going from simple → variable (first variation added), product-level data seeds the first variation.
- A variable product must have ≥ 2 variations — enforced in validation.

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
- At least one product image required.
- Variable product must have ≥ 2 variations.
- No duplicate SKUs within the same product's variations (checked client-side with `indexOf`).
- Simple product: SKU required; if on sale, sale price must be > 0 and < regular price.
- Each variation: SKU required; regular price > 0; if on sale, sale price > 0 and < regular price; each attribute row must have both name and option (and hex value for color); at least one complete attribute required.
- Errors displayed below save/delete buttons.
- Slug uniqueness per brand checked server-side in `saveProduct` before any writes — uses `.neq("id", productId)` so a product updating its own slug doesn't conflict with itself.

---

### Categories

- Only leaf categories are selectable — `getCategoryOptions` uses `collectLeaves` to get leaves only.
- Full path displayed (e.g. `"Sunglasses / Sport / Polarized"`) using `breadcrumbs.join(" / ")` from `LeafEntry` — because two leaves can share the same name and only the path makes them unique. `path` is slug-based (no spaces, lowercase); `breadcrumbs` is display names.
- `id` from `LeafEntry` is the UUID of the leaf node — it's what gets stored in `categoryIds`.

---

### Analytics

- `getTopCategoryViews`: fetches all categories in one query, builds a `catMap` keyed by ID, walks parent chain to build full path. Can't filter to leaves at the query level — all rows needed to walk up the parent chain.
- `buildPath`: walks `parent_id` chain upward, `unshift`ing names to build top-down path.
- Sort and slice in JS after fetching all.
- `RankedList` uses index as key — static server-rendered data with no reordering. `key={r.name}` was a bug: same product name appears multiple times in top sales (once per SKU).

---

### UI Patterns

- `NavProgress` active while saving, uploading, or navigating back. 3px tall with a glow matching the brand accent.
- Save disabled while uploading.
- Category dropdown: max height 320px with scroll.
- Attribute name: select-only dropdown (no free typing). Backdrop div at z-index 9 closes on outside click — the trigger `onClick` just sets the key (never needs to toggle to null) because the backdrop always intercepts the click first when the dropdown is open.
- Price inputs: `step="1"`, `onBlur` reformats to 2 decimal places.
- Placeholder text styled to `#a3a3a3, opacity: 1` globally to override browser default faintness.
- `useParams()` preferred over prop-passing for route params in client components tightly coupled to the route.
- Section order: Description images → Categories → Product images → Variations.

---

### Type Organization

- Shared types (used across ≥ 2 files): `types.ts`.
- File-local types: live in their own file.
- `"use server"` modules cannot re-export types — runtime error.
- `FormImage`, `FormAttr`, `FormVariation`: form-only types, belong in `product-form.tsx`.
- `LeafEntry`: only used alongside `collectLeaves` in `utils.ts`.
- `ProductDetailVariation`: unused, removed.

---

### React Patterns

- `key={index}` is safe for static lists with no reordering. Use stable IDs (e.g. `o.id`) for lists that can reorder or have identity.
- `useRef` for DOM nodes that don't need to trigger re-renders (e.g. hidden file input).
- `useState` initial value only runs on first render.
- `onBlur` vs `onChange` for formatting: `onChange` with `toFixed(2)` would reformat on every keystroke, making the field untyable.
- `Array.prototype.unshift` prepends to front; useful when walking a tree bottom-up to build a path in top-down order.
- `flatMap` maps and flattens one level — equivalent to `map` when the callback returns a non-array, but safe against single-element arrays from Supabase nested joins.
- `indexOf` returns the first occurrence index — `skus.find((s, i) => skus.indexOf(s) !== i)` finds the first duplicate by detecting when the current index doesn't match the first occurrence.
- Supabase: if a query errors, `data` is `null`. Always check the error before using data — `null?.length` is `undefined` (falsy) and silently bypasses checks.
