# Dev Log (July 30, 2026)

## Admin Product Form

### Confirmation Modal
- Fires before save and delete — prevents accidental writes/deletions
- State shape: `{ title, message, proceedLabel, proceedColor, onProceed }`
- `onProceed` holds a function reference (`handleSave` or `handleDelete`); calling it is deferred until the user confirms
- Backdrop is a sibling div (not a parent), same pattern as dropdowns — backdrop at z-index 100, modal box at 101
- Clicking backdrop closes modal; Cancel on the left, Proceed on the right
- Save and Delete buttons disabled during `saving`, `uploading`, and `navigating`

### Slug Mismatch Banner
- On load, compares `product.slug` (DB value) to `slugify(product.name)`
- If different, shows a branded warning banner above the name field
- Save resolves it automatically — slug is computed server-side from name on every save

### Server-Side Slug Computation
- Frontend no longer sends slug; `saveProduct` runs `slugify(input.name.trim())` and writes it
- Slug uniqueness checked against other products in the same brand before writing

### Image Upload Path
- Format: `${safeName}-${crypto.randomUUID()}` — UUID makes collision effectively impossible
- No nested folders; flat structure within the brand's bucket

### Variable Product Pricing
- `sale`, `regularPriceCents` are `null` for variable products — computed from variations instead
- `salePriceCents` saved only if `> 0 AND < regularPrice`, otherwise `null`
- `effectivePrices` uses the sale price when a variation is on sale, otherwise regular price

### Attribute Dropdown Deduplication
- Already-selected attribute names are hidden from other rows in the same variation
- `idx !== ai` excludes the current row so its own name stays visible in the dropdown

### Color Picker
- Rendered conditionally: `{attr.name === "color" && <input type="color" ... />}`
- `value` is always a valid hex when `attr.name === "color"` — initialized to `"#000000"` on color selection

---

## TBYB API

### GET /api/public/packages
- Returns all `tbyb_packages` rows for a brand
- `brands` is a `text[]` column directly on the row — not a join — holds the sunglass brand names in the package (e.g. `['BikerArmour', 'Wiley X']` for combo packages)
- One image per package stored as `image_src` directly on the row — 1:1 so no separate table needed

### POST /api/user/upload
- Authenticated — requires bearer token
- Accepts `multipart/form-data` with `file` and `brandSlug` fields
- Uploads to the brand's Supabase Storage bucket under `tbyb/` folder
- Path format: `tbyb/${safeName}-${crypto.randomUUID()}`
- Uses admin client — RLS is enabled on `storage.objects` with no policies, so the user client would be denied by default
- `getPublicUrl` is synchronous and constructs the URL without a network call — no error to handle
- Upload is decoupled from submission so files are uploaded eagerly as the user picks them, keeping the final submit fast

### POST /api/user/upload (naming)
- Supabase client named `adminSupabase` (not `supabase`) to make it obvious at the call site that RLS is being bypassed

### POST /api/user/tbyb
- Authenticated — `user_id` always populated from JWT
- `packageId` is the UUID from `tbyb_packages.id` — used directly as `tbyb_package_id`, no slug lookup
- Pre-checks package existence with a DB query before insert — FK constraint would also catch it but gives a cryptic Postgres error; the pre-check returns a clean 404
- Package not found returns 404 (not found), missing fields return 400 (bad request) — different status codes for different failure types
- `prescriptionUrl` / `headshotUrl` are URLs from the upload endpoint, sent as optional fields and mapped to `prescription_url` / `headshot_url`
- Returns `{ id }` — the submission UUID so the frontend can display it to the user
- Validation pattern: each field extracted then immediately checked (`const x = body.x; if (!x) return err(...)`) rather than destructuring all at once — makes guard placement explicit
- `!value` used for string checks because `??` misses empty string `""`

---

## Concepts

### HTTP Status Codes
- `400` Bad request — missing or invalid input from the client
- `401` Unauthorized — missing or invalid token
- `404` Not found — resource doesn't exist
- `500` Server error — DB or internal failure
- Distinguishing 400 vs 404 matters: missing field is a client mistake, missing resource is a lookup failure

### RLS (Row Level Security)
- RLS on with no policies = deny all by default, even authenticated users
- Only the service role (admin client) bypasses RLS
- `storage.objects` has RLS enabled — same flag, same behavior as regular tables
- Check with: `select relname, relrowsecurity from pg_class where relname = 'objects' and relnamespace = (select oid from pg_namespace where nspname = 'storage')`
- Storage policies are defined on `storage.objects` with `bucket_id` and path conditions

### Supabase Storage
- `getPublicUrl` is synchronous — constructs URL by string concatenation, no network call, no error to handle
- Equivalent to: `` `${SUPABASE_URL}/storage/v1/object/public/${bucket}/${path}` ``
- Admin client needed when RLS has no permissive policies for the operation

### FK Constraints
- Inserting with an invalid foreign key throws a Postgres error — but the message is cryptic
- A pre-check query before insert gives a clean, readable error response to the frontend

### FormData
- `multipart/form-data` sends binary in labeled parts — binary travels as-is, no base64 encoding
- JSON can't carry binary without base64, which increases size and requires encode/decode
- `req.formData()` is async because it reads the request body stream

### Function References
- `onProceed: handleDelete` stores the function reference — called later when user confirms
- `onProceed: handleDelete()` would call it immediately and store the return value (`void`)
- Arrow function needed when passing arguments: `onClick={() => removeAttr(v.id, ai)}`

### Refs
- `useRef` returns `{ current: <value> }` — stable container, no re-renders on mutation
- Callback ref `ref={(el) => { map.current[key] = el; }}` — React calls it with the element on mount
- Used for `varImgInputs` to map variation IDs to their hidden file inputs

### Nullish Checks
- `!value` catches null, undefined, and empty string `""` — right for required string fields
- `??` only catches null/undefined — misses empty string
- `||` catches all falsy including `0` — too broad for numbers
- `===` checks value and type with no coercion; `==` coerces before comparing

### Image Tables vs JSONB
- Separate `product_images` / `variation_images` tables give on-delete cascade, queryable sort order, and room to add columns
- JSONB array on the row would be simpler for the current use case — no joins, no delete-reinsert dance
- Separate table is the right tradeoff when you need to query images independently or enforce row-level constraints
