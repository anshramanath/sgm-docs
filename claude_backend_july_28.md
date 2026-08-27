# Admin Product Form — Dev Log (July 28, 2026)

## What Was Built

### Confirmation Modal
- Fires before save and delete — prevents accidental writes/deletions
- State shape: `{ title, message, proceedLabel, proceedColor, onProceed }`
- `onProceed` holds a function reference (`handleSave` or `handleDelete`); calling it is deferred until the user confirms
- Backdrop is a sibling div (not a parent), same pattern as dropdowns — backdrop at z-index 100, modal box at 101
- Clicking backdrop closes modal; Cancel button on the left, Proceed on the right

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
- Why FormData: server actions can't receive raw binary — `FormData` uses multipart encoding, which sends binary in labeled parts without base64 overhead

### Variable Product Pricing
- `sale`, `regularPriceCents` are `null` for variable products — computed from variations instead
- `salePriceCents` saved only if `> 0 AND < regularPrice`, otherwise `null`
- `effectivePrices` uses the sale price when a variation is on sale, otherwise regular price

### Attribute Dropdown Deduplication
- Already-selected attribute names are hidden from other rows in the same variation
- `ATTR_SUGGESTIONS.filter(s => !v.attrs.some((a, idx) => idx !== ai && a.name === s))`
- `idx !== ai` excludes the current row so its own name stays visible

### Color Picker
- Rendered conditionally: `{attr.name === "color" && <input type="color" ... />}`
- `value` is always a valid hex when `attr.name === "color"` — `setAttrField` initializes it to `"#000000"` on color selection, and the color input only ever produces valid hex

### Button Disable Logic
- Save and Delete both disabled during `saving`, `uploading`, and `navigating`
- Prevents concurrent operations (e.g. uploading while deleting)

### Miscellaneous
- `sku: input.sku` — server already nulls it for variable products; no need to branch client-side
- Description images use upsert (not insert) — re-saving an unchanged product re-submits existing URLs; upsert on `(brand_slug, src)` is a no-op rather than a constraint violation. Also correct when a shared image picker is added later.
- `removeDescImage` extracted as a named function for consistency with other remove functions
- Placeholder renamed from `"Value"` to `"Option"` to match DB terminology

---

## Concepts

### Refs
- `useRef` returns `{ current: <value> }` — a stable container that doesn't trigger re-renders on mutation
- For a single DOM element: `ref={myRef}` → React sets `myRef.current` to the element on mount
- For multiple elements: callback ref `ref={(el) => { map.current[key] = el; }}` — React calls the function with the element; you store it yourself
- `varImgInputs.current[v.id]?.click()` programmatically opens the file picker for a specific variation

### Function References vs Calls
- `onClick={handleSave}` — passes the function; React calls it on click
- `onClick={handleSave()}` — calls it immediately and passes the return value (`void`) as the handler
- Arrow function needed when passing arguments: `onClick={() => removeAttr(v.id, ai)}`
- `confirm.onProceed` is a value that happens to be a function — you can store it, pass it, or call it with `()`

### React State
- `useState` without a setter: the value is locked for that render; updating requires the setter
- Functional updater `setState(p => ...)`: `p` is the current state — avoids stale closures, needed when new state depends on old state
- `p` is not a copy you can mutate — `p.push(...)` mutates the actual state. `filter`, `map` return new arrays, which is what you assign.

### Arrays and Objects
- `filter`, `map`, `find`, `some` — all non-mutating; they iterate over values and return something new
- `some`: returns `true` if any element matches; stops at the first match
- `find`: returns the first matching element (or `undefined`); stops at the first match
- Spreading an object copies its key-value pairs shallowly — nested objects still share the same reference
- Primitives (strings, numbers, booleans) are copied by value — changing a copy doesn't affect the original

### Nullish vs Falsy
- `?.` short-circuits to `undefined` only for `null` or `undefined` — `0?.toString()` works fine
- `??` falls back only for `null`/`undefined`
- `||` falls back for any falsy value (including `0`, `""`, `false`)
- `===` checks value and type with no coercion; `==` coerces types before comparing

### Supabase
- PostgREST always returns arrays for embedded relations
- `upsert` with `onConflict` specifies which columns to match on; if a row with those values exists, it updates; otherwise inserts
- On delete cascade removes junction rows but not shared table rows (e.g. `description_images` rows survive; only the join is removed)

### HTTP / Requests
- Request structure: method + URL + headers + body
- Multipart: body split into labeled parts separated by a boundary string — each part has its own headers and payload; binary travels as-is
- JSON can't carry binary without base64 encoding, which increases size and requires encode/decode steps

### Misc JS
- `Object.fromEntries(array.map(...))` — builds a lookup object from an array
- `Math.min(...array)` — spread required because `Math.min` takes individual arguments, not an array
- `delete obj.key` on a missing key is a no-op, no error
- Empty array is truthy — check `.length` to test emptiness
- `objects.length` is `undefined` in JS (unlike Python's `len`) — use `Object.keys(obj).length` for objects
