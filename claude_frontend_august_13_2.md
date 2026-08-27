# PrescriptionFramesClient — Learnings & Build Notes (August 13, 2026)

## What Was Built

`src/app/(shop)/rx/PrescriptionFramesClient.tsx` — the client component for the bikershades `/rx?tab=frames` route. Users pick a prescription frame, choose a color, then fill out a 7-step form (steps 0–6) to configure their lenses and submit an order.

### Step flow
- **Step 0** — Optional TBYB order # (to apply deposit toward this order)
- **Step 1** — Prescription (vision type, OD/OS sphere/cylinder/axis)
- **Step 2** — PD (single or dual)
- **Step 3** — Lens material and color/type
- **Step 4** — Coatings (AR, scratch, mirror)
- **Step 5** — Additional info + contact
- **Step 6** — Review and submit

---

## Key Design Decisions

### Frame selection and swatch logic
Three cases when clicking a color swatch:
1. The color is already chosen (chosen frame + chosen color) → deselect both (set frame and pending color to null)
2. Different color on the same or different frame → clear selected frame, set pending color
3. No frame selected yet → set pending color

The "Select Frame" button is disabled until a color is pending for that frame. This enforces that every frame selection has a color attached.

Hover and pending are tracked separately: `hoveredColor` drives the displayed color name while hovering; `pendingColor` is the persisted choice. Hover takes precedence in the display.

### Conflict detection on frame switch
When switching frames, the new frame may have a different rx range. Sphere and cylinder both use the same `sphereOptsForRange(rxLow, rxHigh)` opts. If any saved sphere or cylinder value is not in the new frame's opts, all six prescription fields (OD/OS sphere, cylinder, axis) are reset and the user is sent back to step 1.

`"None"` (sphere or cylinder power of 0) also triggers a conflict if the new frame's range doesn't include 0 — intentional, since "None" represents a real power value.

### Step preservation
When switching frames without a conflict, the current step is preserved. When deselecting a frame, step is also preserved. This avoids forcing users back to step 0 unnecessarily.

### saveToLS always takes explicit values
`saveToLS(toStep, frameId, frameColor, values)` requires the values to be passed explicitly rather than reading from the closure. This matters in `selectFrame` where `setVals(nextVals)` is async — if you called `saveToLS` after `setVals`, the state update wouldn't have applied yet. Passing `nextVals` directly ensures the correct values are saved.

`name` and `email` are stripped before saving to localStorage (they come from auth context, not user input, so no need to persist them).

### LS key isolation
Frames use `${brandSlug}:rx`, TBYB uses `${brandSlug}:tbyb`. No cross-contamination.

### LS not cleared on submission
localStorage is preserved after the user submits so they can refer back or resume if something goes wrong. Clearing it on a Stripe success redirect is the right moment — wired up separately when that redirect is implemented.

---

## Options and Ranges

### sphereOptsForRange
Sphere and cylinder share the same opts array, derived from the frame's `rxLow`/`rxHigh`. Steps down from high to low in 0.25 increments. Loop condition `i >= 0` (not `i > 0`) ensures both endpoints are included.

Floating point is handled by `Math.round(val * 100) / 100` before computing each value, then `toFixed(2)` for display. Without the multiply-then-divide trick, IEEE 754 floating point arithmetic can produce values like `0.30000000000000004` instead of `0.3`. Rounding to the nearest 0.01 (two decimal places) fixes this.

`"None"` is pushed when `raw === 0` (sphere or cylinder power of zero). Positive values get a `+` prefix. Negative values use `toFixed(2)` directly (the minus sign is already there).

### Axis
1–180 (degrees). Disabled when cylinder is null or "None" — axis is only meaningful with a cylinder value.

### PD
- Single: 50–75 in 0.5 steps, plus "None" (we'll follow up)
- Dual (left + right separately): 20–40 in 0.5 steps

### Cylinder
Uses the same `sphereOpts` as sphere — same frame rx range applies to both.

---

## Component Patterns

### PriceLabel
Splits option strings at the first `(` to colorize the price portion in brand color. Uses `indexOf("(")` — simpler and more reliable than regex for this case.

### Dropdown
Controlled by a single `openId` string lifted to the parent. Only one dropdown is open at a time. A full-screen invisible overlay (`fixed inset-0 z-20`) captures outside clicks to close. The overlay's z-index sits below the dropdown list, so clicking an option still works.

### `disabled?` on Dropdown
The `disabled` prop is optional (`disabled?: boolean`). Omitting it gives `undefined` which is falsy — same as passing `false`. Used to disable the axis dropdown when no cylinder is selected.

### Unused destructured variables
When destructuring two values you don't need, name them differently to avoid collision: `const { name: _n, email: _e, ...rest } = values`. Both `_n` and `_e` are intentionally unused; the underscore prefix signals this to the linter.

### key prop
`key` is only meaningful on elements inside a `.map()` array. It's ignored everywhere else. A `key` on a JSX element returned from a plain function call (not inside a map) is noise.

### Optional chaining short-circuit
`a?.b.find(cb)` — if `a` is null/undefined, the entire expression evaluates to `undefined` and the callback `cb` is never invoked. The callback's contents (including any `!` assertions) are safe.

---

## React Concepts Confirmed

### State batching
Multiple `setState` calls in the same synchronous block are batched — React applies them all before re-rendering. The order of calls doesn't matter for the final state.

### Object.is comparison
React bails out of a re-render if the new state is `Object.is`-equal to the old state. Passing the same object reference won't trigger a re-render; always spread to create a new object.

### setStep(s => s + 1)
Functional update form used instead of `setStep(step + 1)` to avoid stale closure issues. Matches the pattern used in TBYBClient.

---

## Pending
- Wire up `/api/user/rx-frame-order` server endpoint (TODO in the step 6 submit block)
- Remove `submitted` state and replace with redirect to Stripe success URL — clear LS there
