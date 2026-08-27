# Sunglass Admin — Dev Log (July 26, 2026)

---

## Features Built

### Confirmation Modal (Save & Delete)
- Modal fires on Save and Delete buttons only — back navigation is unaffected.
- Two patterns for "click outside to close":
  - **Parent div as backdrop + `stopPropagation` on inner box** — inner box is a child, clicks inside bubble up to parent, `stopPropagation` cuts that. Simpler when the overlay is already full-screen.
  - **Sibling backdrop div at lower z-index** — same pattern as dropdowns. Backdrop intercepts outside clicks; the modal box sits above it so clicks inside never reach it. No `stopPropagation` needed. Chosen approach.
- `confirm` state holds `{ title, message, proceedLabel, proceedColor, onProceed }`. All fields explicit — no fallbacks for things we control.

### Slug Sync
- Slug is now computed server-side from `name` in `saveProduct` — client no longer sends it.
- `slug` is still fetched from DB on load.
- On load, if `product.slug !== slugify(product.name)`, a banner shows: "This product's slug is out of sync with the database. Save to resolve." Saving fixes it automatically since the server recomputes.
- Banner styled like the error box — brand accent color, tinted background, solid border.

### Image Upload Path
- All images land flat at the root of the brand bucket — no nested folders.
- Path: `${safeName}-${crypto.randomUUID()}` — UUID makes collision effectively impossible.
- Removed `upsert: true` — with a UUID path, a duplicate is a bug and should throw, not be silently overwritten.

### Not Found Guard Removed
- `notFound()` guard for brand in `ProductDetailPage` removed — admin routes are controlled, brandSlug comes from the layout. Bad URLs crash loudly instead of hiding problems.

---

## JavaScript / TypeScript Concepts

### Equality
- `===` — strict, no type coercion. Always use in TypeScript.
- `==` — coerces types first (`0 == ""` is `true`).
- Coercion = silent type conversion before comparison.

### Nullish / Falsy Operators
- `?.` — optional chaining. Short-circuits to `undefined` if left side is `null` or `undefined`. Anything else (including `0`) proceeds normally.
- `??` — nullish coalescing. Uses right side only if left is `null` or `undefined`.
- `||` — uses right side for any falsy value (`null`, `undefined`, `0`, `""`, `false`).
- Ternary `? :` — branches on any falsy value, same as `||`.

### Primitives vs References
- Primitives (string, number, boolean) are copied by value — changing one variable never affects another.
- Objects and arrays are copied by reference — two variables pointing to the same object share its data. Mutation through one affects the other.
- Only `null` and `undefined` throw when you try to access a property. Primitives like `0` or `""` are auto-boxed to their wrapper object, so `(0).toString()` works. Keying an object with a missing key just returns `undefined`.

### Arrays
- `filter`, `map`, `slice`, `concat` — do not mutate, always return a new array.
- `push`, `pop`, `splice`, `sort` — mutate in place.
- React requires a new array reference to detect state change — always return a new array from state updaters.
- Empty array `[]` is truthy — use `.length` to check if it has elements.
- `Object.keys(obj).length` for counting object keys (plain objects have no `.length`).

### useState
- `useState` without a setter is valid — lock a value across renders without `const x = ...` which would recompute on every render.
- Functional updater `setState(p => ...)` — `p` is the actual current state. Safer than closing over the state variable when multiple updates could be batched.
- TypeScript infers type from the initial value. Empty arrays need explicit annotation (`useState<string[]>([])`). `null` initial values need `useState<T | null>(null)`.

### Destructuring
- Nested: `const { data: { publicUrl } }` — destructures `data` then immediately destructures `publicUrl` from it, all in one statement.
- Array destructuring: `const [value, setter] = useState(...)`.

### Arrow Functions
- Implicit return when body is a single expression with no curly braces: `p => p.filter(...)`.
- With curly braces, explicit `return` is required.

---

## HTTP / Networking

### Request Structure
```
POST /path HTTP/1.1
Host: example.com
Content-Type: application/json

{ "key": "value" }
```
- Request line, headers, blank line, body.
- GET requests have no body.

### Response Structure
```
HTTP/1.1 200 OK
Content-Type: application/json

{ "result": "ok" }
```
- Status line, headers, blank line, body.

### FormData & Multipart
- Server actions only accept serializable data. `File` is binary and can't be JSON-serialized.
- `FormData` uses multipart encoding — body is split into labeled parts separated by a boundary string. Each part has its own headers (filename, content type) and raw bytes.
- No encoding/decoding overhead unlike base64 (~33% size overhead).
- Multipart structure: `------boundary\nContent-Disposition: ...\n\n<raw bytes>\n------boundary--`

---

## Supabase / PostgREST

### Joins
- FK to PK → many-to-one → PostgREST returns an object (or still an array).
- PK to FK → one-to-many → PostgREST always returns an array.
- PostgREST always wraps embedded relations in arrays regardless of uniqueness — it doesn't know from the FK alone that a relation is one-to-one.
- Left join by default — rows on the left are returned even if no matching rows exist on the right (empty array).
- Inner join excludes rows with no match on either side.

### Junction Tables
- Two FKs pointing at two different PKs. Used to link tables that don't directly reference each other (e.g. `product_categories` linking `products` and `categories`).

### `flatMap` on Nested Relations
- `product_description_images` returns `[{ description_images: [{ src, name }] }, ...]`.
- `flatMap(r => r.description_images)` unwraps each inner array and flattens to `[{ src, name }, ...]`.

### `getSession` vs `getUser`
- `getSession` reads from cookie — fast but potentially stale.
- `getUser` hits the Supabase server to verify the token — always fresh.
- Session gives `session.user` with `id`, `email`, `user_metadata`, tokens.

### Storage Upload
- `upload` returns file metadata, not a URL.
- `getPublicUrl(path)` constructs the public URL from the path.
- `upsert: true` suppresses conflict errors — only appropriate when collisions are expected. Remove it when paths are UUID-guaranteed unique so duplicates surface as bugs.
