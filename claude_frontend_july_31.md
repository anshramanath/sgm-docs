# Learnings & Build Log (July 31, 2026)

---

## TBYB Feature

### What was built

- `src/app/(shop)/try-before-you-buy/page.tsx` — server component, fetches user + packages in parallel, passes to client
- `src/app/(shop)/try-before-you-buy/TBYBClient.tsx` — full client component: package grid, 3-step form, file upload, submission
- `src/lib/types.ts` — added `TBYBPackage` and `TBYBSubmission` types
- `src/lib/db/004_tbyb.sql` — `tbyb_packages` + `tbyb_submissions` tables, RLS, seed data
- `src/lib/api.ts` — added `getPackages`, `submitTBYB`, `uploadFile`
- `TBYB_BACKEND_HANDOFF.md` — backend handoff (now stale, endpoints are built)

### Flow

1. Server component fetches `getUser()` + `getPackages()` in parallel
2. Packages rendered in a grid; selecting one auth-gates (redirects to `/sign-in` if not logged in)
3. 3-step form appears inline below the grid
4. On submit: files uploaded in parallel via `uploadFile`, URLs passed into `submitTBYB`
5. `submitTBYB` posts to `/api/user/tbyb` via `authedFetch`

---

## DB Schema — 004_tbyb.sql

### tbyb_packages

| Column | Type | Notes |
|--------|------|-------|
| id | uuid | PK |
| brand_slug | text | FK → brands.slug |
| name | text | Display name |
| slug | text | Unique |
| price_cents | int | Deposit amount |
| image_src | text | Package image path |
| pairs_min | int | |
| pairs_max | int | |
| brands | text[] | Brand names included |
| created_at / updated_at | timestamptz | |

### tbyb_submissions

Submissions denormalize the package — no FK to `tbyb_packages`. Package fields are snapshotted at submission time so historical submissions aren't affected by package changes.

| Column | Type | Notes |
|--------|------|-------|
| id | uuid | PK |
| brand_slug | text | FK → brands.slug |
| user_id | uuid | FK → auth.users, always populated (no guest submissions) |
| package_name / package_slug / package_price_cents / package_image_src / package_pairs_min / package_pairs_max / package_brands | — | Package snapshot |
| od_sphere / od_cylinder / od_axis | text | Nullable |
| os_sphere / os_cylinder / os_axis | text | Nullable |
| lens_type / helmet_size / hat_size / nose_bridge | text | |
| buying_preference | text | maps from `sunglassFit` |
| frame_type | text | |
| special_requests | text | maps from `comments` |
| prescription_url / headshot_url | text | From upload |
| contact_email | text | Not null |
| contact_phone | text | |
| status | text | Not null, no default — backend always passes `'pending'` |
| created_at / updated_at | timestamptz | |

RLS: packages are public read; submissions require auth for select (users read own). Insert goes through admin client on the backend — no insert grant on submissions.

### Why submissions are denormalized

No FK to `tbyb_packages` means historical submissions are unaffected if a package is renamed, repriced, or deleted. The backend receives `packageId`, looks up the package, and snapshots all fields onto the submission.

---

## api.ts — TBYB Functions

### getPackages

- Public, uses `apiFetch`
- `GET /api/public/packages?brandSlug=...`
- Returns `TBYBPackage[]`

### submitTBYB

- Authenticated, uses `authedFetch` (JSON)
- `POST /api/user/tbyb`
- Spreads `TBYBSubmission` + `brandSlug` into body
- 401 → redirect `/sign-in`, 404 → throw, 500/503 → throw, default → redirect `/try-again`

### uploadFile

- Authenticated, uses raw `fetch` — NOT `authedFetch`
- `POST /api/user/upload` with `multipart/form-data`
- Appends `brandSlug` to FormData before sending
- Checks `getUser()` first, redirects to `/sign-in` if not logged in
- Network errors caught in try block; 401 → redirect `/sign-in`, 500 → throw, default → redirect `/try-again`
- Returns `{ url: string }` — callers must key into `.url`

### Why uploadFile doesn't use authedFetch

`authedFetch` hardcodes `Content-Type: application/json` and calls `JSON.stringify(body)`. File uploads need `multipart/form-data` with a browser-generated boundary — setting the header manually breaks the boundary and the server can't parse the parts. Raw `fetch` with a `FormData` body lets the browser set the correct content type automatically.

### Why upload and submission are decoupled

- Upload is slow (storage); submission is fast (DB insert). Combined, submission waits on storage.
- If upload fails inside submission, all form work is lost. Decoupled, only the upload needs to retry.
- Files upload once — if submission fails, the URL is already there and can be retried.

### uploadFile returns { url: string } not string

Returns the full `json.data` object rather than `json.data.url` so callers key into `.url` explicitly, keeping the return shape consistent with the rest of the API response pattern.

---

## Auth — getUser vs getSession vs getToken

### getUser()

Returns the `User` object (id, email, metadata). Validates the session cookie against Supabase's server. Returns `null` if not logged in or token expired. Does NOT expose the token.

### getSession()

Returns the `Session` object which includes `access_token`. Only reads from the cookie — does not validate against Supabase's server. Used in `getToken()` purely to retrieve the token for raw `fetch` requests.

### getToken()

Calls `getSession()` to get the `access_token`. Only needed when making raw `fetch` calls (like `uploadFile`) where you need to manually attach `Authorization: Bearer <token>`. The Supabase client handles the token implicitly for all other calls.

### Two ways to authenticate with Supabase client

**Option 1 — inject token at initialization (what we use):**
```ts
const supabase = createClient(url, anonKey, {
  global: { headers: { Authorization: `Bearer ${token}` } }
});
const { data: { user } } = await supabase.auth.getUser();
```

**Option 2 — pass token directly to getUser:**
```ts
const supabase = createClient(url, anonKey);
const { data: { user } } = await supabase.auth.getUser(token);
```

Both validate the token against Supabase's server. Option 1 is cleaner when you need to make multiple authenticated calls since the token is set once on the client.

---

## TBYBClient — Key Patterns

### Auth gate
Package selection checks `userEmail` prop (passed from server component). If falsy, `router.push("/sign-in")`.

### Custom dropdowns — value vs label
Each option has `value` (what gets stored/sent) and `label` (what the user sees). They differ when the display needs to differ from the stored value — e.g. `value: "plano"`, `label: "None"`. For options where they're the same, `lo()` maps `string[]` to `{ value: s, label: s }`.

### Single `openId` state for dropdowns
One `openId` state controls which dropdown is open. `data-dd` attribute used for outside-click detection via `mousedown` listener.

### Cylinder → axis disable
Axis dropdown is disabled when cylinder is `null` or `"plano"`. On cylinder change to `"plano"`, axis is cleared.

### File upload
Files stored in `FormVals` as `File | null`. On submit, built into a `FormData` with `fd.append("file", file)` and passed to `uploadFile`. Result URL passed as `prescriptionUrl`/`headshotUrl` in submission.

### Error handling
`submitTBYB` and `uploadFile` throw on error. `handleNext` wraps in try/catch, sets `error` state from `e.message`.

### Promise.all for parallel uploads
Files are uploaded in parallel before submission: `Promise.all([uploadFile(rxFd), uploadFile(photoFd)])`. If no file, resolves to `null` and the URL is passed as `null`.

---

## Types

### TBYBPackage
```ts
{
  id: string;
  name: string;
  slug: string;
  priceCents: number;
  pairsMin: number;
  pairsMax: number;
  imageSrc: string;
  brands: string[];
}
```

### TBYBSubmission
```ts
{
  packageId: string;
  odSphere: string | null; odCylinder: string | null; odAxis: string | null;
  osSphere: string | null; osCylinder: string | null; osAxis: string | null;
  lensType: string | null; helmetSize: string | null; hatSize: string | null;
  noseBridge: string | null; sunglassFit: string | null; frameType: string | null;
  comments: string; email: string; phone: string;
  prescriptionUrl: string | null; headshotUrl: string | null;
}
```

---

## Deleted Files

### src/lib/tbyb.ts
Originally held `TBYBSubmission` type + `submitTBYBForm` server action that wrote directly to Supabase. Deleted once the backend endpoint was built:
- `TBYBSubmission` moved to `src/lib/types.ts`
- Server action replaced by `submitTBYB` in `src/lib/api.ts` using `authedFetch`

---

## General Learnings

### "use server" files can only export async functions
Plain arrays/constants exported from a `"use server"` file are treated as server action exports and break at runtime. Move constants to a plain `.ts` file.

### multipart/form-data boundary
When sending a `FormData` body via `fetch`, never manually set `Content-Type`. The browser generates a random boundary string and includes it in the header automatically. Setting it manually omits the boundary and the server can't parse the parts.

### authedFetch vs raw fetch
`authedFetch` is JSON-only (hardcodes `Content-Type: application/json` + `JSON.stringify`). Use raw `fetch` for multipart. Still call `getUser()` first for the auth check.

### redirect() in "use server" functions called from client components
`redirect()` from `next/navigation` works inside server actions called from client components — Next.js propagates the redirect correctly.

### 503 in try/catch vs switch
If a `fetch` call fails at the network level (DNS failure, connection refused), it throws a `TypeError` — no response is returned. In `uploadFile`, which has its own try/catch for the raw `fetch`, there is no synthetic 503, so `case 503` in the switch is unnecessary. In functions using `authedFetch` or `apiFetch`, a synthetic 503 response is constructed in the catch block, so `case 503` is needed.

### generateMetadata .catch(() => null)
`generateMetadata` is called by Next.js separately to generate the page `<title>`. If the data fetch throws (product not found, network error), it would fail metadata generation. `.catch(() => null)` degrades gracefully — metadata falls back to the brand name. The actual 404 handling is in the page component which calls the same function without a catch.

### Fragment with key prop
When `.map()` returns multiple sibling elements, wrap in `<Fragment key={...}>` — shorthand `<>` doesn't accept a `key` prop.

### brands: string[] not string
`brands` on `TBYBPackage` is an array of brand names. Stored as `text[]` in Postgres, returned as `string[]` from the API. Rendered with `.join(", ")` in the client.
