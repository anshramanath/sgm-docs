# Try Before You Buy (TBYB) (August 2, 2026)

## Overview

Customers pick a package, pay a fully-refundable deposit, and receive prescription frames to try at home. They return the frames and place their order — or get their deposit back minus a service fee.

---

## Route

`/try-before-you-buy` — gated to `bikershades` only via `notFound()`. Other brands get a 404.

---

## Files Changed

| File | Role |
|------|------|
| `src/app/(shop)/try-before-you-buy/page.tsx` | Server page, brand gate, Suspense skeleton, passes `email` + `name` to client |
| `src/app/(shop)/try-before-you-buy/TBYBClient.tsx` | All client-side UI and state |
| `src/lib/types.ts` | `TBYBPackage`, `TBYBSubmission` types |
| `src/lib/api.ts` | `getPackages`, `submitTBYB`, `uploadFile` |
| `src/lib/db/004_tbyb.sql` | Migration: `tbyb_packages` + `tbyb_submissions` tables |
| `src/lib/db/initial_schema.sql` | Consolidated schema reflecting 001–004 |
| `src/components/layout/Navbar.tsx` | TBYB badge in desktop nav (bikershades only) |
| `src/components/layout/HeaderIcons.tsx` | TBYB badge in mobile nav as its own bordered section |

---

## User Flow

1. Land on page → see all packages as cards
2. Click **Select Package**:
   - Not signed in → redirect to `/sign-in`
   - Signed in → package selected, 3-step form opens
   - Click same package again → deselects
3. **Step 1** — Prescription (OD/OS sphere, cylinder, axis)
4. **Step 2** — Preferences (lens type, helmet size, hat size, nose bridge, buying preference, frame type)
5. **Step 3** — Additional info (comments, prescription upload, headshot upload) + Contact info (name, email, phone)
6. Submit → **Step 4** (success screen) with submission ID

---

## State Design

- `step: number` (1–4) — step 4 is the success screen, no separate `submitted` boolean
- Selecting a package resets to step 1 automatically (`setStep(1)` in `selectPkg`)
- `vals` — flat form state object initialized with `email` and `name` from the server (pre-populated from user account)
- `submitting` — disables Back, Submit, and Select Package during API call
- `rxUploading` / `photoUploading` — disables Back and Submit during file uploads

---

## Key Behaviors

**Axis clearing** — `update()` clears `odAxis`/`osAxis` when cylinder is set to `"None"`:
```ts
if (key === "odCylinder" && value === "None") next.odAxis = null;
if (key === "osCylinder" && value === "None") next.osAxis = null;
```

**Null coercion at network boundary** — `submitTBYB` coerces optional null fields to `"None"` before POST:
```ts
odAxis: data.odAxis || "None",
osAxis: data.osAxis || "None",
comments: data.comments || "None",
phone: data.phone || "None",
prescriptionUrl: data.prescriptionUrl || "None",
headshotUrl: data.headshotUrl || "None",
```

**Auth guard** — `selectPkg` checks `!email || !name` client-side for UX. Even if bypassed, `authedFetch` calls `getUser()` server-side independently — unauthenticated submissions return 401 and never hit the DB.

**File uploads** — prescription and headshot are uploaded separately via `POST /api/user/upload` before submission. The upload URL is stored in state and passed to `submitTBYB`.

---

## DB Schema

### `tbyb_packages`
| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid | PK |
| `brand_slug` | text | FK → brands |
| `name` | text | Display name |
| `slug` | text | `slugify(name)` — unique |
| `price_cents` | int | Deposit amount |
| `image_src` | text | Supabase storage URL |
| `pairs_min` | int | |
| `pairs_max` | int | |
| `brands` | text[] | Brand names shown in "Includes" |

**Slug convention:** `slugify(name)` — lowercase, spaces/special chars → `-`, no leading/trailing dashes. The `+` in combo package names (e.g. `BikerArmour + 7Eye`) becomes `-`, yielding `bikerarmour-7eye`.

### `tbyb_submissions`
Snapshots the full package details at submission time (denormalized) so price/package changes don't affect existing submissions. Key contact fields: `contact_name`, `contact_email`, `contact_phone`. RLS: users can only read their own rows.

---

## Packages (Bikershades)

| Name | Slug | Deposit | Pairs | Brands |
|------|------|---------|-------|--------|
| BikerArmour | `bikerarmour` | $229 | 3–5 | BikerArmour |
| Wiley X | `wiley-x` | $249 | 3–5 | Wiley X |
| 7Eye | `7eye` | $249 | 3–5 | 7Eye |
| BikerArmour + Wiley X | `bikerarmour-wiley-x` | $279 | 5–8 | BikerArmour, Wiley X |
| BikerArmour + 7Eye | `bikerarmour-7eye` | $279 | 5–8 | BikerArmour, 7Eye |
| BikerArmour + 7Eye + Wiley X | `bikerarmour-7eye-wiley-x` | $329 | 7–10 | BikerArmour, 7Eye, Wiley X |
| 7Eye Ziena | `7eye-ziena` | $349 | 3 | 7Eye |

Images stored at: `https://zgcekcoatiskqbdruadg.supabase.co/storage/v1/object/public/bikershades/packages/<filename>.webp`

---

## Nav Badges

Both desktop and mobile show a TBYB badge (brand color text, white background, brand color ring via `box-shadow`) above the Sale badge. Bikershades only.

- **Desktop** (`Navbar.tsx`): inline `<li>` before Sale in the nav list
- **Mobile** (`HeaderIcons.tsx`): own `div` with `border-t` separator above the Sale section

---

## Backend Notes

- `POST /api/user/tbyb` — expects all `tbyb_submissions` fields including `contact_name`
- `POST /api/user/upload` — multipart, returns `{ url: string }`
- `GET /api/public/packages` — returns `tbyb_packages[]` filtered by `brandSlug`
- Null sentinel: optional fields are sent as `"None"` (not null) to satisfy DB `NOT NULL` constraints
- `contact_name` column was added via: `alter table tbyb_submissions add column contact_name text not null;` (table was empty at time of migration)
