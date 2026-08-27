# Learnings & Build Notes (July 29, 2026)

## Auth Flow

### How sign-in/sign-up works
- All auth logic lives in `src/lib/auth.ts` as server actions (`"use server"`)
- Server actions can write cookies **only when called from a client component** — if called from a server component they run inline and cannot write cookies. This is why `AuthForms.tsx` is a client component even though it calls server-side auth functions.
- Sign-up sends a confirmation email with `emailRedirectTo` pointing to `/sign-in?email=<encoded>`. The sign-in page reads that query param and shows a "Email confirmed — sign in." notice, pre-filling the email field.
- Duplicate email detection: Supabase returns `data.user?.identities?.length === 0` for already-registered emails (no error is thrown).

### Forgot password / reset password
- `requestPasswordReset` calls `supabase.auth.resetPasswordForEmail` with `redirectTo: brand.url/reset-password`
- Supabase sends a link containing a `code` query param (not a hash fragment — it's a real search param, readable server-side)
- Reset page (`/reset-password`) reads `code` from searchParams, passes it to `ResetForm` (client component)
- `ResetForm` calls `resetWithCode(code, password)` — a single server action that does `exchangeCodeForSession` then `updateUser`. No separate callback route needed.
- If exchange fails (expired/reused link), Supabase returns an error and we show "invalid or expired" message via the `error` query param on the reset page.

### PKCE (Proof Key for Code Exchange)
- Supabase uses PKCE under the hood for the reset flow. A `code_verifier` is stored in cookies on the origin that initiated the request, and the `code_challenge` is sent to Supabase.
- When the user clicks the reset link and lands on the same origin, the verifier is present and the exchange works.
- If the user somehow landed on a different origin (different brand), the verifier would be missing and the exchange would fail — by design.
- The code exchange opens a short-lived session. If `updateUser` fails after the exchange, the user is technically logged in for that session — so we call `signOut()` on failure.

### Input validation
- Frontend validates before calling the server action (blocks unnecessary network round trips, gives instant feedback).
- Server action validates again (frontend can always be bypassed — direct API calls, curl, etc.).
- Validation rejects bad input. Sanitization cleans/transforms it (e.g. trimming whitespace, encoding HTML). We do validation, not sanitization.
- `type="text"` on email inputs instead of `type="email"` — suppresses the browser's native validation tooltip so our custom error messages are the only feedback shown.
- Email regex: `/^[^\s@]+@[^\s@]+\.[^\s@]+$/` — checks for non-whitespace chars, an `@`, a dot, and a domain part. `.test(email)` runs the regex against the string.

### Email enumeration protection
- Password reset always returns the same success response regardless of whether the account exists — prevents attackers from probing which emails are registered.

### Cookies vs localStorage
- Both are scoped per origin (`protocol + domain + port`). `localhost:3000` and production are completely separate.
- Cookies can be read/written server-side; localStorage is browser-only.
- Supabase uses cookies (via `@supabase/ssr`) so the server can read the session on every request.

---

## Supabase / SMTP

### Custom SMTP with Resend
- Resend free tier restricts sending to verified domains only — you cannot send to arbitrary emails until you own and verify a domain.
- SMTP settings: Host `smtp.resend.com`, Port `465`, Username `resend`, Password = Resend API key, Sender email = `onboarding@resend.dev` (Resend's shared domain for testing).
- Once a real domain is purchased and verified with Resend, you can send to anyone.

### Supabase redirect URLs
- Site URL: the primary brand's domain (used as base for auth emails).
- Additional redirect URLs: add each brand's domain + `**` wildcard suffix so Supabase allows redirects from all brand subpaths.
- `emailRedirectTo` and `resetPasswordForEmail redirectTo` must use `getBrand().url` (not a `.env` var) because the redirect is brand-dependent at runtime.

---

## Database

### Migration file pattern
- `initial_schema.sql` = the full source of truth (all migrations merged). Run this on a fresh DB.
- `001_core_catalog.sql`, `002_...`, `003_...` = incremental migrations for existing DBs.
- Rule: when adding a new numbered migration, also update `initial_schema.sql` to include it. `initial_schema = 001 + 002 + 003 + ...`

### Join table vs array column
- `product_categories` is a join table (not an array in `products`) because:
  - Referential integrity — deleting a category cleans up the link automatically (cascade)
  - Query in both directions efficiently (find products in a category, find categories for a product)
  - Arrays in Postgres have no FK support — no cascade, no integrity guarantees

### ON DELETE CASCADE
- When a parent row is deleted, all child rows referencing it are automatically deleted.
- Used throughout: deleting a brand cascades to products, variations, images, etc.
- `admins` table references `auth.users(id) on delete cascade` — deleting an auth user removes their admin record.

### FK nullability
- A FK and `NOT NULL` are separate constraints. A nullable FK (`col uuid references other(id)`) allows `NULL` — meaning "no reference." If a value is provided it must reference a valid row. `ON DELETE CASCADE` only fires for non-null values.
- `ALTER TABLE cart_items ALTER COLUMN product_id DROP NOT NULL;` — removes the NOT NULL requirement while keeping the FK constraint.

---

## Next.js

### Layouts don't re-render on navigation
- `layout.tsx` components persist across navigations — they never re-render when the URL changes.
- Server Components in layouts render once on initial load and are frozen after that.
- Client Components in layouts only re-render when their own state changes, not on URL changes.
- Regular page components (not in layout) re-render on every navigation — this is normal Next.js behavior, not special.

### Server Components and cookies
- Server actions called from client components can write cookies (the response is a real HTTP response).
- Server actions called from server components run inline — no separate HTTP response, so cookies cannot be written.

### searchParams in pages
- `searchParams` in Next.js 15 page components is a `Promise` — must be `await`ed.
- Hash fragments (`#...`) are never sent to the server — browser-only. Supabase uses query params (`?code=...`) for the reset code specifically so the server can read it.

---

## Try Before You Buy (TBYB) — Design

### Feature overview
- Bikershades-only feature. Customers try prescription frames at home before lenses are made.
- Customer selects a package (brand combination + deposit), fills out a form (prescription + preferences + uploads), submits.
- Shop fulfills by sending frames, customer returns them and orders lenses.

### Architecture decisions
- TBYB does **not** go through the normal cart (`cart_items.product_id NOT NULL` would break it, and adding nullable branching throughout cart code is messy for a single-brand feature).
- Separate tables: `tbyb_packages` (the 7 deposit options) and `tbyb_submissions` (one row per customer form submission).
- `tbyb_packages.brand_slug` FK to `brands` — scopes packages to bikershades, supports other brands in future with no schema changes.
- `tbyb_submissions.tbyb_package_id` FK to `tbyb_packages` — traces every submission back to the chosen package.
- `user_id` on submissions is nullable (`on delete set null`) to allow guest submissions.
- Payment method (Stripe vs manual email link) TBD — owner needs to decide. Stripe Payment Links are a low-code option (no custom checkout) if on-site payment is wanted.

### Proposed schema
```sql
create table tbyb_packages (
  id          uuid primary key default gen_random_uuid(),
  brand_slug  text not null references brands(slug) on delete cascade,
  name        text not null,
  slug        text not null unique,
  price_cents int  not null,
  image_src   text not null,
  pairs_min   int  not null,
  pairs_max   int  not null
);

create table tbyb_submissions (
  id               uuid primary key default gen_random_uuid(),
  tbyb_package_id  uuid not null references tbyb_packages(id) on delete cascade,
  user_id          uuid references auth.users(id) on delete set null,
  od_sphere text, od_cylinder text, od_axis text,
  os_sphere text, os_cylinder text, os_axis text,
  lens_type text, helmet_size text, hat_size text,
  nose_bridge text, buying_preference text, frame_type text,
  special_requests text,
  prescription_url text,
  headshot_url     text,
  status           text not null default 'pending',
  created_at       timestamptz not null default now()
);
```
