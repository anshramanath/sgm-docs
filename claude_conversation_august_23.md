# Domain Migration & Email Setup — Summary (August 23, 2026)

Migration of three e-commerce sites (ProSport Sunglasses, BikerShades, Sunglass Monster) from SiteGround/WordPress to Vercel, with full DNS and transactional email reconfiguration.

## Sites migrated
| Domain | Site host | Email |
|---|---|---|
| prosportsunglasses.com | Vercel | Google Workspace + spam filter |
| bikershades.com | Vercel | Google Workspace + Resend/SES |
| sunglassmonster.com | Vercel | Spam filter (Google behind it) + Resend/SES |

---

## DNS Concepts

- **Registrar** (GoDaddy) — where the domain is owned/purchased. Controls who the domain belongs to.
- **Nameserver** — the actual server that answers "where does this domain point?" Can be different from the registrar (e.g. domain bought at GoDaddy, DNS managed at SiteGround).
- **A record** — maps a domain/subdomain to an IP address. Controls where a *website* lives.
- **MX record** — controls where *email* gets delivered. Has a priority (lower number = tried first).
- **CNAME record** — an alias; "this name is really just another name" (e.g. `www` → apex domain).
- **TXT record** — arbitrary text, used for verification (Google site verification), and email authentication:
  - **SPF** — declares which servers are allowed to send mail for the domain.
  - **DKIM** — cryptographic signature proving outgoing mail wasn't tampered with.
  - **DMARC** — policy telling receivers what to do if SPF/DKIM checks fail.

Only **one** valid TXT record per name/type should exist (e.g. only one `_dmarc` record) — duplicates cause unpredictable behavior. GoDaddy often auto-creates a default `_dmarc` record that must be *edited*, not duplicated, when migrating.

## Migration pattern used (per domain)

1. Read the full DNS record list from SiteGround's Site Tools → DNS Zone Editor (get untruncated values by clicking "Edit" on each row).
2. Add the site's own domain to Vercel (Settings → Domains → Add Existing), get the required A record (`216.198.79.1`).
3. In GoDaddy, edit the apex A record to point to Vercel's IP.
4. Switch nameservers from SiteGround to GoDaddy defaults (`ns.domaincontrol.com`) — this makes GoDaddy authoritative, so **all** original mail records must be manually recreated there or email breaks.
5. Recreate every MX, CNAME (DKIM), TXT (SPF/DMARC/Google verification), and subdomain A record from SiteGround into GoDaddy, matching values exactly (character-for-character — DKIM keys and SPF strings are especially easy to break with a small copy error).
6. Watch for GoDaddy's default `_dmarc` record conflicting with the real one — edit it in place rather than adding a duplicate.
7. Wait for SSL cert generation on Vercel (can take a few minutes), then confirm the site loads.

## Resend (transactional email) setup

Used for auth emails (signup confirmation, password reset) via Supabase Auth's custom SMTP.

- Added domain in Resend → Manual setup → got 3 DNS records: `TXT resend._domainkey` (DKIM), `MX send` (priority 10, `feedback-smtp.us-east-1.amazonses.com`), `TXT send` (SPF, `v=spf1 include:amazonses.com ~all`).
- These live on a `send.` subdomain — no conflict with the domain's existing apex SPF/DKIM.
- Added to GoDaddy, verified in Resend dashboard (Domain added → DNS verified → Domain verified).
- **"Enable Receiving" was left off** — that's for inbound mail routing and would conflict with the domain's real inbox (Google Workspace); only outbound transactional mail was needed.

### Supabase SMTP configuration
- Host: `smtp.resend.com`
- Port: `465`
- Username: `resend`
- Password: Resend API key (with sending access)
- Sender email: `noreply@sunglassmonster.com`
- Sender name: brand display name

## Auth email templates

Rebuilt Supabase's default (plain, unstyled) "Confirm signup" and "Reset Password" emails as branded HTML (black/white minimal style, styled button, plain-link fallback, expiry notice).

- Subject line kept brand-agnostic: **"Confirm your email address"** — since one Supabase project serves three different brands, a template can't hardcode one brand's name in copy.
- **`{{ .Data.brand }}`** used instead of `{{ .SiteURL }}` for the visible brand name in the header/footer — because `SiteURL` is a single fixed project-level value and can't vary per domain, but `.Data` reads from that specific user's `user_metadata`.
- `brand: getBrand().name` added to the `data` object in the `signUp()` call so it's written into `user_metadata` at signup.
- Because Supabase's `.Data` variable always reads from the user's *stored* `user_metadata` (not from whatever specific auth call triggered the email), `{{ .Data.brand }}` also works correctly in the Reset Password email — even though `resetPasswordForEmail()` has no `data` parameter itself.
- **Caveat:** only works for users who signed up *after* the `brand` field was added to the code — existing users' metadata won't retroactively have it.
- Contact link uses a fixed shared inbox (`mailto:help@sunglassmonster.com`) for all three brands rather than a brand-specific URL, since that was simpler and matches how support is actually handled.

## Bug found & fixed: confirmation links falling back to Vercel URL

**Symptom:** clicking "Confirm email address" redirected to `https://prosport-sunglasses-frontend.vercel.app/?code=...` instead of the real domain, and dropped the `?email=` param.

**Cause:** Supabase's **Site URL** (Authentication → URL Configuration) was still set to the raw Vercel preview domain, and the **Redirect URLs** allow-list only contained `.vercel.app` domains — not the real custom domains. Since `emailRedirectTo` (set per-brand via `getBrand().url`) didn't match anything in the allow-list, Supabase silently discarded it and fell back to the Site URL.

**Fix:**
- Changed **Site URL** to a real domain (`https://bikershades.com`).
- Added all three real domains to **Redirect URLs** with wildcards: `https://prosportsunglasses.com/**`, `https://bikershades.com/**`, `https://sunglassmonster.com/**` (plus `localhost:3000` for local dev).

## Confirmation code flow (PKCE)

- `{{ .ConfirmationURL }}` in the email is a **Supabase-hosted** verification link, not the site directly. Supabase verifies the token server-side first, then redirects to `emailRedirectTo` with a `code` param appended.
- For **password reset**, that `code` is required — `resetWithCode()` calls `exchangeCodeForSession(code)` then `updateUser({ password })`.
- For **signup confirmation**, the email is already confirmed server-side by the time the redirect happens — the `code` param is only needed if you want to auto-sign-in the user immediately; otherwise it can be ignored and the user just logs in normally afterward.

## Outstanding / next steps
- Test signup + password reset end-to-end on all three domains post-fix, using a *newly created* test account (old accounts won't have `brand` in metadata).
- Confirm `{{ .Data.brand }}` renders the correct brand name per domain.
- Once all three domains confirmed working (site + email), SiteGround hosting can be canceled.

---

## Stripe

### Branding assets (Icon / Logo)
Stripe's requirements: ≥128×128px, square, JPG or PNG, **max 512KB**.
- A 1024×1024 PNG logo came in at ~1,130KB — over the limit.
- Fixed by palette-quantizing to 256 colors (`Image.quantize(colors=256, method=Image.FASTOCTREE)`) rather than resizing — kept full 1024×1024 resolution, brought it to ~234KB, no visible quality loss (illustration was mostly flat colors, not a photo, so quantization is nearly lossless here).
- Lossless `optimize=True` alone only saved ~12% (1130KB → 997KB) — not enough on its own; palette reduction did the real work.

### Business name shown at checkout ("Shiao Co" vs. "Sunglass Monster")
- Stripe **Account name** (Settings → Account details) was already correctly set to "Sunglass Monster."
- The mismatch traced to **Business details** being incomplete — "Shiao Co" was showing as a fallback/legal name while business info hadn't been fully filled in.
- Distinction: **Legal business name** (actual registered entity, used for compliance/tax) vs. **DBA / public business name** (what customers see on checkout and statements) — these can differ. Filling out "Add business information" should surface a DBA field to set the public name independently of the legal one.
- Caught a near-miss: a *different* Stripe account ("tatalo sandbox," unrelated to this project) was open in another tab with a "Close account" button visible — confirmed account identity (via the account ID in the URL) before doing anything, since closing an account is irreversible.

### Test mode → Live mode checklist
Two separate credential sets, both scoped per-mode (not just a toggle):
1. **API keys** — test (`sk_test_...` / `pk_test_...`) vs. live (`sk_live_...` / `pk_live_...`) are different values; live ones must replace test ones wherever stored (e.g. Vercel env vars).
2. **Webhook signing secret** — webhook endpoints are configured separately per mode in Developers → Webhooks. A new endpoint must be created under live mode (same production URL, same events), which generates its own `whsec_...` secret — the test-mode secret will not verify live-mode webhook signatures.

Steps when going live: create live webhook endpoint → copy new signing secret → update `STRIPE_WEBHOOK_SECRET` env var → update secret/publishable keys to live versions → test with a real small transaction to confirm both payment and webhook fire correctly.
