# Learnings & Build Log (July 26, 2026)

---

## Auth Flow

### Sign Up
- `signUp` calls Supabase with `emailRedirectTo: ${getBrand().url}/sign-in?email=...`
- Email is URL-encoded (`encodeURIComponent`) so `@` doesn't break the query string — Next.js decodes it automatically on read
- Supabase silently returns 200 for duplicate emails (email enumeration protection) — detect via `data.user.identities.length === 0`
- On success: switch to sign-in mode, show notice with the email used

### Sign In
- Server action called from a client component (`AuthForms.tsx`) — required for cookie writes
- `"use server"` functions called from server components run inline and cannot write cookies
- `"use server"` functions called from client components become true POST requests and CAN write cookies
- This is why all auth triggers live in client components even though the logic is server-side

### Forgot Password
- User enters email → `requestPasswordReset` server action → Supabase sends reset email
- Always show the same generic notice regardless of whether the email exists — prevents email enumeration (same principle as RLS: never confirm or deny whether a resource exists)
- `redirectTo` uses `getBrand().url` so the link always goes to the correct brand's production URL
- Requesting from localhost and clicking the link on production breaks PKCE — test reset from production only

### Reset Password
- Supabase reset email lands on `/reset-password?code=...` (PKCE flow — code is a search param, not a hash)
- Server page reads `code` from `searchParams`, passes it to `ResetForm` client component as a prop
- On form submit, `resetWithCode(code, password)` server action exchanges the code for a session then immediately calls `updateUser`
- If `updateUser` fails, sign out to prevent a dangling session
- Codes are single-use — using the same reset link twice gives "invalid flow state" error
- Confirm password field validates client-side before any server call
- Should redirect away if user is already logged in (same as `/sign-in`)

---

## PKCE (Proof Key for Code Exchange)

Originally designed for OAuth (e.g. MCP auth), also used by Supabase for password reset.

**Flow:**
1. Generate `code_verifier` (secret, kept locally in cookies/storage)
2. Hash it → `code_challenge` (public, sent to server upfront)
3. Server stores the challenge, emails you a `?code=` link
4. On redemption: send the `code` + original `code_verifier`
5. Server hashes the verifier and checks it matches the stored challenge — proves same party initiated the flow

**Why it's secure:** Intercepting `?code=` from the URL is useless without the verifier. The challenge is public and meaningless on its own.

**Cross-device breaks it:** Verifier lives in browser A's cookies. If the email link opens in browser B, verifier is missing → `access_denied`. This is why localhost→production mismatch fails.

**Supabase handles all of this under the hood** — you just call `resetPasswordForEmail` and `exchangeCodeForSession`.

---

## Storage: Cookies vs localStorage

- Both are scoped per **origin** (`protocol + domain + port`) — same-origin policy
- `localhost:3000` and `https://brand.vercel.app` are completely separate storage contexts
- Two tabs on the same origin share the same localStorage and cookies
- **localStorage**: browser-only, never sent to server
- **Cookies**: sent to the server with every request — this is how session auth works
- Requesting a password reset from one brand and redeeming it on another would fail — the code verifier cookie belongs to the initiating origin

---

## Input Validation

**Two layers — frontend blocks pointless round-trips, server is the safety net (client validation can always be bypassed):**

Frontend (`AuthForms.tsx` — `validate()` closes over component state, no params needed):
- Non-empty email
- Valid email format via regex `/^[^\s@]+@[^\s@]+\.[^\s@]+$/`
- Non-empty name (signup only)
- Non-empty password, minimum 8 characters

Server (`auth.ts`):
- Same checks on `signIn`, `signUp`, `resetWithCode`
- `validateEmail` and `validatePassword` helper functions

**Regex breakdown** `/^[^\s@]+@[^\s@]+\.[^\s@]+$/`:
- `^` start, `[^\s@]+` = one or more chars that aren't whitespace or `@`, `\.` = literal dot, `$` end
- `.test(string)` returns true/false — negate with `!` to produce an error condition

**Input sanitization** = cleaning/transforming input (e.g. `.trim()`, lowercasing). Validation rejects; sanitization fixes. No aggressive sanitization needed here since input goes to Supabase (parameterized) and nothing is rendered as raw HTML.

Browser's native `type="email"` tooltip was replaced with `type="text"` so our custom validation messages show instead.

---

## Supabase Config

### URL Configuration
- **Site URL**: fallback redirect — set to primary brand's production URL
- **Redirect URLs allowlist**: `https://<brand>.vercel.app/**` for each brand — `/**` wildcard covers all paths
- Covers email confirmation, password reset, and any future OAuth redirects

### Custom SMTP (Resend)
- Supabase free tier has a low global rate limit on auth emails (shared across all email types)
- Custom SMTP removes this limit — Supabase routes through your provider instead
- Resend SMTP: host `smtp.resend.com`, port `465`, username `resend`, password = API key (`re_...`)
- Without a verified domain, Resend only sends to your own signup email address
- `onboarding@resend.dev` is Resend's shared sender for unverified accounts
- Once you own a domain, verify it in Resend → can send to any recipient

### Admin Table Cascade
- `admins` table FK to `auth.users` needs `on delete cascade` — without it, deleting a user in the Supabase dashboard fails
- Migration: `alter table admins drop constraint admins_user_id_fkey, add constraint admins_user_id_fkey foreign key (user_id) references auth.users(id) on delete cascade;`
- `001_core_catalog.sql` already includes the cascade

---

## DB Migrations Structure

```
001_core_catalog.sql       — brands, categories, products, images, admins, view functions
002_user_cart_bookmarks.sql — cart_items, bookmarks (with RLS)
003_orders.sql             — orders, order_items, sales triggers (with RLS)
initial_schema.sql         — full source of truth (001 + 002 + 003 merged)
```

**Rule:** `initial_schema - 002 - 003 = 001` and `001 + 002 + 003 = initial_schema`

**Workflow:** When adding a new migration (e.g. `004_something.sql`), also update `initial_schema.sql` to incorporate the changes — keeps it as the complete source of truth for fresh DB setups.

---

## Schema Design

### Join Table vs Array for Product Categories
`product_categories` is a join table (not an array column on `products`) because:
- **Many-to-many**: a product can belong to multiple categories, a category has many products — join table handles both directions efficiently
- **Referential integrity**: FK on each `category_id` means the DB rejects invalid references; an array has no FK constraint
- **Cascade deletes**: deleting a category automatically removes its `product_categories` rows — with an array you'd manually clean up stale IDs

### On Delete Cascade
- Defined on FK columns: `references parent(id) on delete cascade`
- When a parent row is deleted, all child rows referencing it are automatically deleted
- Without it, deleting a parent either fails (FK violation) or leaves orphaned rows

---

## Route Groups

- `(groupName)` in Next.js strips the segment from the URL — purely organizational, no URL impact
- `(shop)/(footer)` nests footer pages inside the shop layout without changing paths
- Imports are unaffected — Next.js resolves routes by file path not URL

---

## Products: Simple vs Variable

- **Simple product**: `sku` non-null at product level — one SKU, one price
- **Variable product**: `sku` null at product level, each variation has its own SKU
- Use `!!product.sku` to detect simple products — not `variations.length` (search endpoint doesn't return variations)
- Swatches deduplicate by slug — two variation names resolving to the same slug show as one swatch (intended)

---

## Order Types

```ts
type Order = {
  status: "processing" | "shipped" | "refunded";
  refundedCents: number | null;  // non-null + status !== "refunded" = partial refund
  carrier: string | null;
  trackingNumber: string | null;
};
```

- Partial refund derived: `refundedCents > 0 && status !== "refunded"`
- Status colors: processing = `#737373`, shipped = brand color, refunded = `#000000`

---

## Key Patterns

### Cookie Writes
Only work in: Route Handlers, Server Actions called from client components.
Do NOT work in: Server Component render functions (even calling a `"use server"` function from a server component doesn't help — it runs inline in the same render context).

### Email Enumeration
Never confirm or deny whether an account/resource exists. Always return the same response. Same principle as RLS: the attacker learns nothing from the response.

### `getBrand().url` vs `window.location.origin`
- `getBrand().url` always points to production — use for emails sent to users
- `window.location.origin` reflects current environment — use when the redirect must match where the request came from
