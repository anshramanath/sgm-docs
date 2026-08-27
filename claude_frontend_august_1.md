# Try Before You Buy — Build Notes (August 1, 2026)

## What Was Built

A multi-step form flow for requesting prescription frame try-ons. Users pick a package, pay a refundable deposit, fill out prescription and fitting info, then submit. The deposit is collected separately after confirmation.

---

## Commit History (relevant range)

| Commit | What happened |
|--------|--------------|
| `TRY BEFORE YOU BUY MA BOIS` | Initial React port from the HTML prototype |
| `cleaning up nicer` | Refactored options, file upload UX, scroll, validation, error banner, submission ID, Opt type removed |
| `anon tbyb insert` | Added Supabase insert grant + RLS policy for authenticated users |
| `api md update` | Updated API.md to reflect "None" sentinel, correct option strings |
| `drop and initial` | Merged 004_tbyb.sql into initial_schema.sql; updated drop_schema.sql |
| *(uncommitted)* | Suspense + skeleton, email prop rename, sphere/cylinder ±8 and positive-first order, package toggle deselect, padding moved to page, buttons disabled during upload |

---

## From HTML Prototype → React

The HTML prototype (`try-before-you-buy (2).html`) was a static proof-of-concept with inline JS (`TBYB` object). The React port changed several things:

### `Opt = { value, label }` → `string[]`

The prototype kept value and label separate (e.g. `{ value: "plano", label: "None" }`). In React, value === label for every option, so the type was collapsed to plain `string[]` and the `lo()` helper and `Opt` type were deleted.

### `"plano"` → `"None"`

The prototype used `"plano"` as the internal value for no cylinder/sphere. Changed to `"None"` — which is also what gets stored in the DB as the sentinel string.

### Dropdown close mechanism

- Prototype: `document.addEventListener('click', e => { if (!e.target.closest('[data-dd-wrap]')) ... })`
- React: `{openId && <div className="fixed inset-0 z-20" onClick={() => setOpenId(null)} />}` — a transparent overlay that captures the click and closes whatever is open.

### Cylinder → Axis enable/disable

- Prototype: imperative `ddOnChange` callbacks wired after DOM insertion.
- React: `axisDisabled` computed directly from `vals.odCylinder === "None"` on every render — no event wiring needed.

### File upload

- Prototype: drag-and-drop (`ondragover`, `ondrop`) + click. Filename shown as a chip.
- React: click-only (`<input type="file">` + `<label>`). No drag-and-drop. "Remove" button lives inline in the filename chip. Added `uploading` boolean prop to show an "Uploading…" badge while the upload is in flight.

### Upload timing

- Prototype: files uploaded at submit time via `Promise.all([uploadFile(...), uploadFile(...)])` inside the submit handler.
- React: upload-on-select via `useEffect([vals.rxFile])` and `useEffect([vals.photoFile])`. By the time the user hits Submit, the URLs are already in state (`rxUrl`, `photoUrl`). This is why clearing the file sets the URL back to `null` — the old URL would otherwise persist and get submitted.

### Step validation

- Prototype: no validation before advancing — just `this.step++`.
- React: `handleNext` validates the current step before calling `setStep(s => s + 1)`.

### Error banner

- Prototype: imperatively injected a `div#submitError` after `#stepMount` on failure.
- React: `{error && <div>}` in JSX with the same visual styling (red-50 bg, red-200 border, circle-i icon). Cleared on Back, cleared on `handleNext` entry.

### Submission ID

- Prototype: mocked with `Array.from({length:8}, () => '0123456789ABCDEF'[...]).join('')`.
- React: uses the real `result.id` from the API response, shown as `.slice(-8).toUpperCase()`.

### Scroll

- Prototype: `window.scrollTo({ top: formTop - 90, behavior: 'smooth' })` called synchronously inside `selectPackage`, using a `getBoundingClientRect()` on `#formRegion`.
- React: `useEffect([selectedPkg])` with `formRef.current?.scrollIntoView({ behavior: "smooth", block: "start" })`. The ref is on a sentinel `<div>` at the bottom of the package grid (always in DOM). The `useEffect` fires after render — which matters because the form section is conditionally rendered (`{selectedPkg && ...}`), so it doesn't exist in the DOM when `selectPkg` fires.

---

## Architecture

### Suspense Streaming (`page.tsx`)

```
TBYBPage (sync)         → renders header immediately
  └── Suspense
        ├── fallback: TBYBSkeleton   → shown while data loads
        └── TBYBLoader (async)       → awaits getUser() + getPackages(), then renders TBYBClient
```

Only async components trigger Suspense. `TBYBLoader` exists solely to be the async boundary — you can't make `TBYBPage` itself async while keeping the sync header visible.

The skeleton matches the exact card layout: 8 cards, same padding/heights as real content. 8 was chosen because 7 would look odd when more packages are added.

### Padding ownership

Bottom padding (`pb-20 lg:pb-28`) lives on the `TBYBPage` wrapper div, not inside `TBYBClient`. The client renders both the package grid section and the form section — both had their own bottom padding, which caused extra space when the form was present. Moving it to the page makes it a fixed constant regardless of form state.

### Email prop

```ts
// Before: optional
{ packages, userEmail }: { packages: TBYBPackage[]; userEmail?: string }

// After: required string, at least ""
{ packages, email }: { packages: TBYBPackage[]; email: string }
```

Caller: `<TBYBClient packages={packages} email={user?.email ?? ""} />`

The `?? ""` handles the unauthenticated case. The prop is always a `string`, so the component doesn't need to handle `undefined`. Initialization: `useState<FormVals>({ ...INIT, email })` — spread order means `email` overrides `INIT.email`.

---

## Form

### Steps

| Step | Fields |
|------|--------|
| 1 | Prescription — OD (right), OS (left): Sphere, Cylinder, Axis |
| 2 | Fitting — Lens Type, Helmet Size, Hat Size, Nose Bridge, Sunglass Fit, Frame Type |
| 3 | Additional Info — Comments, Rx upload, Headshot upload, Email, Phone |

### Option ranges

Sphere and cylinder both go to ±8.00 diopters in 0.25 steps (32 steps × 0.25 = 8.00). Order is positive → None → negative (top of dropdown = most positive). This is the reading order preferred on the form.

### Validation (`handleNext`)

```
Step 1: sphere required for both eyes; cylinder required; axis required only if cylinder ≠ "None"
Step 2: all 6 fitting fields required
Step 3: email.trim() required; blocks if rxUploading || photoUploading
```

### Package toggle

`selectPkg` checks `selectedPkg?.id === pkg.id` first — if the same package is clicked, it sets `selectedPkg(null)` and returns early. The form section disappears. No form reset is needed since the form only appears when `selectedPkg` is truthy.

### Button states

| Condition | Back | Submit/Continue |
|-----------|------|-----------------|
| Normal | visible (invisible on step 1) | enabled |
| `rxUploading \|\| photoUploading` | disabled (opacity-30) | disabled (opacity-50) |
| `isPending` (submitting) | disabled | disabled, shows "Submitting…" |

---

## "None" Sentinel Pattern

DB uses `NOT NULL` on all columns. Optional fields use `"None"` as a string sentinel.

- **Client state**: holds `null` for unset optional fields.
- **`submitTBYB` in `api.ts`**: coerces at the network boundary, only for truly optional fields:
  ```ts
  comments: data.comments || "None",
  phone: data.phone || "None",
  prescriptionUrl: data.prescriptionUrl || "None",
  headshotUrl: data.headshotUrl || "None",
  ```
- **Required fields** (`packageId`, `email`, all prescription/fitting fields): must arrive non-falsy — frontend validation ensures this before submit.

---

## DB (`004_tbyb.sql` / `initial_schema.sql`)

- All `tbyb_submissions` columns `NOT NULL`
- `user_id` references `auth.users(id)` with `ON DELETE SET NULL` — submission survives if user is deleted
- RLS: authenticated users can `SELECT` their own rows and `INSERT` their own rows
- Package details (name, slug, price, brands) are snapshotted on the submission at insert time — not joined live

---

## React Concepts Reinforced

### `setState` is asynchronous

Queues a re-render; doesn't update the variable in place. Reading state right after `setState` gives the old value. Use updater form (`setStep(s => s + 1)`) when the new value depends on the old.

### `useEffect` dependency checking

Compares dep array values after each render. If a non-state variable changes without triggering a re-render, the effect never fires. Only state and props reliably cause re-renders (and thus dependency checks).

### Scroll timing

`useEffect` fires after the render — that's why scroll belongs there. The form section (`{selectedPkg && <section>}`) doesn't exist in the DOM when `selectPkg` fires. By the time `useEffect([selectedPkg])` runs, the render has completed and the form is mounted.

### Clearing a URL when a file is removed

```ts
if (!vals.rxFile) { setRxUrl(null); return; }
```

When the user removes a file (`rxFile → null`), the effect fires and clears `rxUrl` so the previously-uploaded URL doesn't get submitted with the form.

### Spread order

```ts
{ ...INIT, email }  // email overrides INIT.email — last key wins
```

---

## File Map

| File | Role |
|------|------|
| `src/app/(shop)/try-before-you-buy/page.tsx` | Sync shell — Suspense, skeleton, async loader |
| `src/app/(shop)/try-before-you-buy/TBYBClient.tsx` | All client state, form steps, package grid |
| `src/lib/api.ts` → `submitTBYB` | "None" coercion at network boundary |
| `src/lib/api.ts` → `uploadFile` | Multipart file upload (server action) |
| `src/lib/types.ts` → `TBYBSubmission` | Client-facing type (nullable optional fields) |
| `src/lib/db/004_tbyb.sql` | Standalone migration — schema, RLS, seed |
| `src/lib/db/initial_schema.sql` | Consolidated full schema (includes TBYB tables) |
| `src/lib/db/drop_schema.sql` | Dev teardown — drops TBYB tables + functions |
| `API.md` | Backend contract — "None" sentinel, correct option strings documented |
| `try-before-you-buy (2).html` | Original HTML prototype (reference only) |
