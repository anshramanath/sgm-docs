# Learnings & Build Log (July 22, 2026)

## Auth Flow

### Sign Up
- `signUp` calls Supabase with `emailRedirectTo: ${getBrand().url}/sign-in?email=...`
- Email is URL-encoded (`encodeURIComponent`) so `@` doesn't break the query string — Next.js decodes it automatically on read
- Supabase silently returns 200 for duplicate emails (email enumeration protection) — detect via `data.user.identities.length === 0`
- On success: switch to sign-in mode, show notice with the email used

### Sign In
- Server action called from client component (`AuthForms.tsx`) — this is required for cookie writes
- `"use server"` functions called from server components run inline and cannot write cookies
- `"use server"` functions called from client components become true POST requests and CAN write cookies
- This is why all auth triggers live in client components even though the logic is server-side

### Forgot Password
- User enters email → `requestPasswordReset` server action → Supabase sends reset email
- Always show the same generic notice regardless of whether the email exists (prevents email enumeration)
- `redirectTo` uses `getBrand().url` so the link always goes to the correct brand's production URL

### Reset Password
- Supabase reset email lands on `/reset-password?code=...` (PKCE flow — code is a search param, not a hash)
- Server page reads `code` from `searchParams`, passes it to `ResetForm` client component
- On form submit, `resetWithCode(code, password)` server action (called from client) exchanges the code for a session then immediately calls `updateUser`
- If `updateUser` fails, sign out to prevent dangling session
- Confirm password field validates client-side before any server call

### PKCE (Proof Key for Code Exchange)
- Before the reset request: generate `code_verifier` (secret, stored locally), hash it → `code_challenge` (public, sent to server)
- Server stores the challenge. On redemption, you send the code + verifier. Server hashes verifier and checks it matches the stored challenge
- Intercepting the `?code=` from the URL is useless without the verifier
- Supabase handles all of this under the hood — you just call the auth functions
- **Cross-device breaks PKCE**: verifier lives in browser A's storage, but if the email link opens in browser B, verifier is missing → `access_denied`
- Each origin (`protocol + domain + port`) has its own localStorage/cookies — `localhost:3000` and production are completely separate

### Email Confirmation
- Uses OTP link (not PKCE) — Supabase verifies server-side and redirects to `emailRedirectTo` with no code exchange needed
- `emailRedirectTo` embeds the email as a query param so the sign-in page can pre-fill it and show "Email confirmed" notice
- After sign-up, switch to sign-in mode and show notice rather than redirecting

---

## Supabase Config

### URL Configuration
- **Site URL**: fallback redirect, set to primary brand's production URL
- **Redirect URLs allowlist**: add `https://<brand>.vercel.app/**` for each brand — `/**` wildcard covers all paths
- These cover both email confirmation and password reset redirects

### Custom SMTP (Resend)
- Supabase free tier has low email rate limits — custom SMTP removes this
- Resend SMTP: host `smtp.resend.com`, port `465`, username `resend`, password = API key
- Without a verified domain, Resend only sends to your own signup email
- `onboarding@resend.dev` is Resend's shared sender for unverified accounts
- Once you own a domain, verify it in Resend and swap the sender email

### Admin Table
- `admins` table needs `on delete cascade` on the `auth.users` FK — without it, deleting a user in the Supabase dashboard fails
- Migration: `alter table admins drop constraint admins_user_id_fkey, add constraint admins_user_id_fkey foreign key (user_id) references auth.users(id) on delete cascade;`

---

## Route Groups

- `(groupName)` in Next.js strips the segment from the URL — purely organizational, no URL impact
- `(shop)/(footer)` nests footer pages inside the shop layout without changing paths
- Imports are unaffected since Next.js resolves routes by file path, not URL

---

## Products: Simple vs Variable

- **Simple product**: has a `sku` at the product level — one SKU, one price
- **Variable product**: `sku` is null at product level, variations each have their own SKU
- Use `!!product.sku` to detect simple products (not `variations.length` — search endpoint doesn't return variations)
- Swatches deduplicate by slug — two variation names that resolve to the same slug show as one swatch (intended behavior)

---

## Order Types

```ts
type Order = {
  status: "processing" | "shipped" | "refunded";
  refundedCents: number | null;  // non-null + status !== "refunded" = partial refund
  carrier: string | null;
  trackingNumber: string | null;
  ...
};
```

- Partial refund derived: `refundedCents > 0 && status !== "refunded"`
- Status colors: processing = `#737373`, shipped = brand, refunded = `#000000`

---

## Shipping Language

- Changed from "Free shipping" to "Shipping included in the price" everywhere
- Rationale: returns for non-seller-error are refunded minus shipping cost, so shipping isn't truly "free"

---

## Footer Pages (Hardcoded for Sunglass Monster)

Contact info lives directly in pages — not in `brand.ts` (which is one-for-all brands):
- Email: `help@sunglassmonster.com`
- Phone: `877-245-3721`
- Hours: Mon–Fri · 9am–4pm (CST)

---

## Key Patterns

### Cookie Writes
Only work in: Route Handlers, Server Actions called from client components.  
Do NOT work in: Server Component render functions (even if the called function has `"use server"`).

### RLS / Email Enumeration
Same principle: never confirm or deny whether a resource/account exists. Return the same response either way.

### `getBrand().url` vs `window.location.origin`
- `getBrand().url` always points to production — use for emails sent to users
- `window.location.origin` reflects current environment — use when the redirect must match where the request came from
