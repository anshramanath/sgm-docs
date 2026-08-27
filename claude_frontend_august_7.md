# TBYB — Final Build Notes (August 7, 2026)

## saveToLS Refactor

### Signature

```ts
function saveToLS(toStep: number, pkgId: string | null)
```

Both params required — explicit at every call site, no stale closure surprises.

### Call sites

| Where | Call | Why |
|---|---|---|
| `selectPkg` — new selection | `saveToLS(step, pkg.id)` | Package id available synchronously before React applies state |
| `selectPkg` — deselect | `saveToLS(step, null)` | Only place that can null out packageId in LS |
| Step advances (1→2, 2→3, 3→4) | `saveToLS(step + 1, selectedPkg!.id)` | Save destination step before React applies it |

### Why saveToLS was removed from the `selectedPkg` useEffect

Two reasons:

1. **Deselect case** — when the user clicks a selected package, `setSelectedPkg(null)` fires. The effect is guarded by `if (selectedPkg && ...)` so it never runs, meaning `packageId` in LS would never be nulled. Moving to `selectPkg` handles both select and deselect explicitly.

2. **Deliberateness** — save should only happen on explicit user action, not as a side effect of state changes that can fire on mount or re-render.

### Why saveToLS is NOT called in handleRxFile / handlePhotoFile

Uploads can fail. The URL only lands in `vals` after a successful `update("rxUrl", r.url)`. The next `saveToLS` call (step advance or package selection) will pick it up. Saving mid-upload risks persisting incomplete state.

---

## TBYB Success Page

Route: `/try-before-you-buy/success`

```tsx
useEffect(() => {
  try {
    const stored = localStorage.getItem(`${brand.slug}:tbyb`);
    if (!stored) return;
    const data = JSON.parse(stored);
    data.packageId = null;
    localStorage.setItem(`${brand.slug}:tbyb`, JSON.stringify(data));
  } catch {}
}, []);
```

Nulls `packageId` on mount — form data (prescription, fitting, contact) persists for next visit, but user has to re-select a package. No timer needed (unlike order success) because there's no cart sync on mount — it's a fire-and-forget LS write.

Success URL in TBYBClient: `${window.location.origin}/try-before-you-buy/success`  
Cancel URL: `${window.location.origin}/try-before-you-buy`

---

## TBYB History (Account Page)

- Unpaid submissions filtered out at the call site: `submissions.filter(s => s.status !== "Unpaid")`
- Unpaid = pre-payment, user never completed checkout — no point showing it

---

## Mobile Upload Fix

Next.js server actions have a 1MB default body size limit. Mobile camera photos are typically 3–10MB, causing silent failures. Fixed in `next.config.ts`:

```ts
experimental: {
  serverActions: {
    bodySizeLimit: "10mb",
  },
},
```

---

## Error / 404 / Try Again Pages

`mb-[20vh]` was pulling content above center on all three pages. Removed from `error.tsx`, `not-found.tsx`, and `try-again/page.tsx` — all now truly vertically centered.
