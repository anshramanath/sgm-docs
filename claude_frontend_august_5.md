# TBYB (Try Before You Buy) — Build Notes (August 5, 2026)

## What Was Built

A multi-step checkout flow (`TBYBClient.tsx`) that collects prescription, fitting, and contact info, uploads files to Supabase, then creates a Stripe checkout session for the package deposit.

Steps: **1** Prescription → **2** Fitting → **3** Additional Info + uploads → **4** Review → Stripe

---

## State Design

### `FormVals` holds everything serializable

```ts
type FormVals = {
  odSphere, odCylinder, odAxis,
  osSphere, osCylinder, osAxis,
  lensType, helmetSize, hatSize,
  noseBridge, sunglassFit, frameType,
  comments, name, email, phone,
  rxUrl: string | null,    // upload result URL, not the File object
  photoUrl: string | null,
};
```

`rxUrl`/`photoUrl` live in `vals` rather than separate state so they're automatically included in localStorage serialization.

`File` objects can't go in localStorage — storing the URL instead is the correct pattern.

---

## File Upload Design

### Direct functions instead of `useEffect`

The original pattern used `useEffect` watching `vals.rxFile`. The problem: on mount, the effect fires with `null`, clobbering any URL restored from localStorage.

The fix: remove file from state entirely, pass files directly into upload functions.

```ts
async function handleRxFile(f: File | null) {
  if (!f) { update("rxUrl", null); return; }
  setRxUploading(true);
  try {
    const fd = new FormData(); fd.append("file", f);
    const r = await uploadFile(fd);
    update("rxUrl", r.url);
  } catch {
    setError("Prescription upload failed!");
  } finally { setRxUploading(false); }
}
```

**Why this is better:** upload only happens on an explicit user action (file picker change), never on mount. No footgun.

### `FileUpload` component

- Shows checkmark icon when `url` is set, upload icon otherwise
- When uploaded: shows label as a link (opens file in new tab) + Remove button
- When empty: "Browse A File"
- Labels: "Prescription Uploaded" / "Headshot Uploaded"

---

## localStorage Persistence

Key: `${brandSlug}:tbyb`

```ts
function saveToLS(toStep: number) {
  const { name: _name, email: _email, ...serializableVals } = vals;
  localStorage.setItem(`${brandSlug}:tbyb`, JSON.stringify({
    packageId: selectedPkg!.id,
    step: toStep,
    vals: serializableVals,
  }));
}
```

`name` and `email` are excluded because they come from the auth session on load — no point persisting them.

`toStep` is required (not optional) so every call site is explicit about what step it's saving. `setStep` is async — passing the value directly avoids reading stale state from the closure.

### Restore on mount

```ts
useEffect(() => {
  const stored = localStorage.getItem(`${brandSlug}:tbyb`);
  if (!stored) return;
  const { packageId, step: savedStep, vals: savedVals } = JSON.parse(stored);
  const pkg = packages.find(p => p.id === packageId);
  if (pkg) setSelectedPkg(pkg);
  if (savedStep) setStep(savedStep);
  if (savedVals) setVals(prev => ({ ...prev, ...savedVals }));
}, []);
```

Restore is a plain `setStep(savedStep)` — no step mapping needed because `saveToLS` always writes the correct post-advance step. Refreshing at any point takes you back to exactly where you were.

### When `saveToLS(toStep)` is called

- On package selection (via `selectedPkg` useEffect) — `saveToLS(step)`, saves current step with the new package so refresh mid-step restores both
- On each step advance (1→2, 2→3, 3→4) — `saveToLS(step + 1)`, saves the destination step before React applies the state update

### On success (future work)

Set `packageId: null` in localStorage. This preserves all form fields (prescription, fitting, contact) but forces the user to re-select a package next visit.

---

## Scroll Behavior

```ts
useEffect(() => {
  if (selectedPkg && formRef.current) {
    saveToLS(step);
    formRef.current.scrollIntoView({ behavior: "smooth", block: "start" });
  }
}, [selectedPkg]);
```

Fires on both:
- Mount, when a package is restored from localStorage
- User clicking a package card

`saveToLS(step)` here saves the current step with the selected package — primary save point for package selection, so a refresh before any step advance still restores the correct package.

`selectPkg` no longer calls `setStep(1)` — switching packages mid-form keeps the current step.

---

## Auth Flash Fix

`loggedIn` starts as `null` (undetermined) instead of `false` (confirmed logged out).

```ts
const [loggedIn, setLoggedIn] = useState<boolean | null>(null);
```

```ts
const isActuallySignedIn = isSignedIn && (loggedIn ?? true);
```

`?? true`: when undetermined, trust the server prop (`isSignedIn`). This prevents the navbar from briefly showing "Sign In" on load for authenticated users.

---

## Schema Changes

Added to `tbyb_submissions`:

```sql
refunded_cents    int,
shipping_address  jsonb,
```

`shipping_address` is `null` until the Stripe webhook fires after payment — Stripe collects it at checkout, no frontend collection needed.

---

## API Changes

- Post-payment status: `Processing` → `Curating`
- Webhook now stores `shipping_address` from Stripe session on `checkout.session.completed`
- `refundedCents` added to submission record response
- `cancelUrl` for TBYB Stripe session: `/try-before-you-buy` (not `/`)

---

## Pending

- Success page: null out `packageId` in localStorage so form data persists but package clears
- Account page: display TBYB submission history using `getSubmissions()`
