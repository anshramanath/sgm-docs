# Session Learnings (August 13, 2026)

## What Was Built

`PrescriptionFramesClient.tsx` — the client component for the bikershades `/rx?tab=frames` page. A multi-step prescription frame ordering form.

### Features
- Frame grid with color swatch selection
- 7-step form: TBYB order #, prescription, PD, lens material & color, coatings, additional info, review
- Local storage persistence across sessions (key: `${brandSlug}:rx`)
- File uploads for prescription and headshot
- Step dots progress indicator
- Review step summarizing all selections
- Submitted confirmation screen

---

## State Model

| State | Type | Purpose |
|---|---|---|
| `selectedFrame` | `PrescriptionFrame \| null` | Which frame's form is open |
| `pendingColor` | `{ frameId, colorSlug } \| null` | Which swatch is clicked (persists pre and post form open) |
| `hoveredColor` | `{ frameId, colorSlug } \| null` | Which swatch the mouse is over |
| `step` | `number` | Current form step (0–6) |

`selectedColor` is **derived**, not stored: `selectedFrame?.colors.find(c => c.slug === pendingColor?.colorSlug)`.

### Swatch Click Logic
Three cases:
1. **Same frame + same color (isChosen)** — deselect: clear both `pendingColor` and `selectedFrame`
2. **Different frame or color, form open** — clear `selectedFrame`, set new `pendingColor`
3. **No frame selected** — set new `pendingColor` (clearing `selectedFrame` is a no-op)

"Select Frame" button is disabled until a color is pending on that frame.

---

## Key Design Decisions

- **`pendingColor` vs `selectedFrame`**: `pendingColor` tracks the chosen swatch at all times. `selectedFrame` is only set when "Select Frame" is clicked — this is what opens the form. Deselecting the chosen swatch clears both.
- **No `selectedColor` state**: derived from `selectedFrame` + `pendingColor` to avoid sync bugs.
- **`step + 1` / `s => s + 1`**: step advancement uses relative increments, not hardcoded target steps — matches the TBYB pattern.
- **Separate LS keys**: TBYB uses `${brandSlug}:tbyb`, frames uses `${brandSlug}:rx` — no collision.
- **Frame switch preserves step**: selecting a different frame no longer resets to step 0.

---

## Sphere/Cylinder Options Generation

```ts
function sphereOptsForRange(rxLow: number, rxHigh: number): string[] {
  const steps = Math.round((rxHigh - rxLow) / 0.25);
  const opts: string[] = [];
  for (let i = steps; i >= 0; i--) {
    const raw = Math.round((rxLow + i * 0.25) * 100) / 100;
    if (raw === 0) opts.push("None");
    else if (raw > 0) opts.push("+" + raw.toFixed(2));
    else opts.push(raw.toFixed(2));
  }
  return opts;
}
```

- `steps` = number of gaps between rxLow and rxHigh in 0.25 increments
- Loop is `i >= 0` (not `i > 0`) because `i = 0` gives rxLow — needed for inclusivity (fence post problem)
- `Math.round((rxHigh - rxLow) / 0.25)` — outer round needed because floating point error can make an integer result like `23.9999...`
- `Math.round(raw * 100) / 100` — inner round eliminates floating point error at the hundredths place before the `=== 0` check
- `toFixed(2)` — ensures consistent display (`"2.00"` not `"2"`) for whole numbers after dividing back by 100
- The rx range applies to **both** sphere and cylinder (same opts used for both dropdowns)

---

## Floating Point

Binary floating point (IEEE 754) can't represent most decimals exactly — same issue in JavaScript, Python, Java, C, and most other languages.

**Why**: numbers are stored in binary, and decimals like `0.1` have no exact binary representation, just like `1/3` has no exact decimal representation.

**Effect**: `0.1 + 0.2 === 0.30000000000000004` in JS.

**Fix**: multiply by a power of 10 to shift the decimal place into integer territory, `Math.round`, then divide back.

```js
Math.round(value * 100) / 100  // round to 2 decimal places
```

**Why not just `Math.round`**: rounds to the nearest integer, destroying decimals entirely (`Math.round(0.25) === 0`).

**Python display**: Python hides floating point error by printing the shortest string that rounds back to the same float — so `0.5 + 0.5` prints `1.0` even though the internal representation may not be exact.

---

## Misc React / TypeScript Notes

- **`key` on list items**: required in `.map()` so React can track which element is which across re-renders
- **`?? null`**: coerces `undefined` to `null` for TypeScript — `e.target.files?.[0]` is `File | undefined`, `onChange` expects `File | null`
- **`setSelectedFrame(null)` when already null**: React compares with `Object.is`, sees no change, skips re-render — true no-op, no guard needed
- **`selectedFrame &&` as form guard**: TypeScript narrows `selectedFrame` to non-null inside the block, eliminating need for `!` assertions
- **Optional chaining `?.`**: returns `undefined` (not an error) if the left side is nullish
