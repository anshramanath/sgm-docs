# Dev Log (July 31, 2026)

## Admin Product Form

### Confirmation Modal
- Fires before save and delete — prevents accidental writes/deletions
- State shape: `{ title, message, proceedLabel, proceedColor, onProceed }`
- `onProceed` holds a function reference; calling it is deferred until the user confirms
- Backdrop is a sibling div — backdrop at z-index 100, modal box at 101
- Save and Delete buttons disabled during `saving`, `uploading`, and `navigating`

### Slug Mismatch Banner
- On load, compares `product.slug` (DB value) to `slugify(product.name)`
- Shows a branded warning banner if different; save resolves it automatically

### Server-Side Slug Computation
- `saveProduct` runs `slugify(input.name.trim())` — frontend never sends slug
- Slug uniqueness checked against other products in the same brand before writing

### Image Upload Path
- Format: `${safeName}-${crypto.randomUUID()}` — UUID makes collision effectively impossible
- Flat structure within the brand's bucket

### Variable Product Pricing
- `sale`, `regularPriceCents` are `null` for variable products — computed from variations
- `salePriceCents` saved only if `> 0 AND < regularPrice`, otherwise `null`

### Attribute Dropdown Deduplication
- Already-selected attribute names hidden from other rows in the same variation
- `idx !== ai` excludes the current row so its own name stays visible

### Color Picker
- Rendered conditionally: `{attr.name === "color" && <input type="color" ... />}`
- `value` always valid hex when `attr.name === "color"` — initialized to `"#000000"` on color selection

---

## TBYB API

### Schema — `004_tbyb.sql`

**`tbyb_packages`**
- Stores package definitions: `name`, `slug`, `price_cents`, `image_src`, `pairs_min`, `pairs_max`, `brands` (text[])
- `brands` is a text array of sunglass brand names in the package — e.g. `['BikerArmour', 'Wiley X']` for combo packages
- `image_src` is a full Supabase public URL under `{brandSlug}/packages/`
- Has `created_at` / `updated_at`
- RLS enabled, no policies — all reads go through admin client

**`tbyb_submissions`**
- Full snapshot of package data at submission time: `package_name`, `package_slug`, `package_price_cents`, `package_image_src`, `package_pairs_min`, `package_pairs_max`, `package_brands`
- No FK to `tbyb_packages` — snapshot means submission survives package deletion/rename
- Tied to brand via `brand_slug` with `ON DELETE CASCADE`
- `user_id` nullable with `ON DELETE SET NULL` — submission survives user deletion
- `contact_email` is `NOT NULL` — enforced at both DB and route level
- `status` is `NOT NULL`, no default — always provided by the route
- RLS: `grant select to authenticated` + policy scoped to `auth.uid() = user_id`

### GET /api/public/packages
- Returns all packages for a brand, camelCased
- Admin client used — RLS on packages has no policies so user client would be denied

### POST /api/user/upload
- Authenticated — requires bearer token
- Accepts `multipart/form-data` with `file` and `brandSlug`
- Uploads to `{brandSlug}/packages/{safeName}-{uuid}` in Supabase Storage
- Admin client used — RLS on `storage.objects` is enabled with no policies
- `getPublicUrl` is synchronous — constructs URL by string concatenation, no network call, no error to handle
- Upload decoupled from submission so files upload eagerly as user picks them — final submit is just a fast DB insert
- Named `adminSupabase` at call site to make RLS bypass obvious

### POST /api/user/tbyb
- Authenticated — `user_id` always from JWT, never from body
- Validates `brandSlug`, `email`, `packageId` — each checked immediately after retrieval
- Looks up package from DB and stores full snapshot — frontend price/name can't be trusted
- Returns `{ id }` — submission UUID for the frontend to display
- `!value` used for string checks — catches empty string `""` which `??` misses

---

## Concepts

### HTTP Status Codes
- `400` Bad request — missing or invalid input
- `401` Unauthorized — missing or invalid token
- `404` Not found — resource doesn't exist (distinct from 400 which is a client input error)
- `500` Server error — DB or internal failure

### RLS vs Grants
- **Grants** — standard Postgres permissions, always enforced. Control whether a role can perform an operation on a table at all. Default: nothing granted to anyone except the owner.
- **RLS** — a toggle on top of grants. When on with no policies, denies all rows to everyone. Policies are what grant row-level access.
- Both must pass for a query to succeed. Service role (admin client) bypasses both.
- `anon` role is used before login; JWT switches the effective role to `authenticated`
- `auth.uid()` returns null for anon — a policy like `using (auth.uid() = user_id)` returns zero rows for unauthenticated requests even if they have the grant

### Supabase Storage and RLS
- `storage.objects` has RLS enabled — same flag, same behavior as regular tables
- No policies on storage = deny all by default; admin client needed
- `getPublicUrl` constructs URL without a network call — equivalent to `` `${SUPABASE_URL}/storage/v1/object/public/${bucket}/${path}` ``

### FK Behavior
- `ON DELETE CASCADE` — deleting the referenced row deletes dependent rows
- `ON DELETE SET NULL` — deleting the referenced row nulls the FK column; dependent row survives
- No `ON DELETE` clause — deleting the referenced row errors if any FK still points to it
- FK can be nullable; PK cannot

### Snapshot Pattern
- Order items and TBYB submissions store a copy of the data at the time of the transaction
- Means historical records are unaffected by product/package renames, reprices, or deletions
- Tradeoff: data can drift from the source of truth, but for financial/fulfillment records that's correct behavior
- FK still useful as a soft reference for triggers (orders) or admin joins — but not required when the snapshot has everything needed

### Function References
- `onProceed: handleDelete` — stores the function, called later
- `onProceed: handleDelete()` — calls it immediately, stores return value (`void`)
- Arrow function needed when passing arguments: `onClick={() => fn(id)}`

### Nullish Checks
- `!value` catches null, undefined, and `""` — right for required string fields
- `??` only catches null/undefined — misses empty string
- `||` catches all falsy including `0`
- `===` checks value and type with no coercion; `==` coerces first

### Refs
- `useRef` returns `{ current: <value> }` — stable container, no re-renders on mutation
- Callback ref `ref={(el) => { map.current[key] = el; }}` — React calls it with the element on mount
- Used for `varImgInputs` to map variation IDs to their hidden file inputs
