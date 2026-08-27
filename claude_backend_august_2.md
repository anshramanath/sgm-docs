# TBYB — Build + Learnings (August 2, 2026)

## What Was Built

### Admin TBYB Page (`/admin/[brandSlug]/tbyb`)

A full admin view for Try Before You Buy submissions, bikershades-only.

**Files:**
- `src/lib/admin/tbyb.ts` — `getTbybSubmissions`, `updateTbybStatus` server actions (admin client, requireAdmin gate)
- `src/app/admin/[brandSlug]/tbyb/page.tsx` — server component, fetches and passes data down
- `src/app/admin/[brandSlug]/tbyb/tbyb-table.tsx` — client component with all interactivity
- `src/lib/types.ts` — `TbybSubmission` type
- `src/components/sidebar.tsx` — TBYB nav item, bikershades-only conditional

**Table features:**
- Filter tabs (All, Processing, Emailed, Curating, Shipped, Received) with counts
- Expandable rows showing: Contact, Package, Status dropdown + Save, Prescription grid, Special requests, Preferences, Uploads
- Status dropdown with draft/save pattern — save only enabled when selection differs from DB value
- Links for prescription/headshot: `!== "None"` check (not truthiness) since values are always a URL or the string "None"

---

## Security

### Why adminSupabase for tbyb_submissions insert
Authenticated users holding the anon key + a valid JWT can hit Supabase directly, bypassing the route. If `authenticated` had insert grants, they could insert fake `package_price_cents`. Using adminSupabase in the route means the route is the only door — it fetches the real package from the DB before inserting, so price can't be spoofed.

### Grants vs RLS
- **Grants** — table-level access control (can this role touch this table at all?)
- **RLS policies** — row-level access control (which rows can this role see/modify?)
- Both must pass. Admin client bypasses RLS but not column constraints (NOT NULL, CHECK).
- `USING` — filters rows for SELECT/UPDATE/DELETE (what rows can you read/touch?)
- `WITH CHECK` — validates the new row state for INSERT/UPDATE (is the row you're writing valid?)
- UPDATE uses both: USING to find the row, WITH CHECK to validate the result.

### What authenticated gets
- `orders`, `order_items`, `tbyb_submissions` — SELECT only (read own rows via RLS)
- `cart_items`, `bookmarks` — full CRUD (user preference data, no financial constraint)
- No insert on orders/submissions — prevents price manipulation via direct Supabase calls

---

## Database

### Snapshot pattern
`tbyb_submissions` stores a full copy of package data at insert time (name, price, pairs, brands, image). This means submissions survive package edits or deletions. Same pattern as `order_items`.

### NOT NULL as atomic validation
Every field in `tbyb_submissions` is NOT NULL (except user_id). A missing field causes the entire insert to fail — nothing partial is written. Optional fields use the string `"None"` as a sentinel so columns stay NOT NULL.

### Cascade delete and triggers
- `order_items` has `ON DELETE CASCADE` from `orders` — deleting an order removes its items.
- The `decrement_total_sales_on_refund` trigger is `AFTER UPDATE ON orders`, not `AFTER DELETE` — so hard-deleting an order does NOT decrement total_sales. Triggers are operation-specific.

### undefined inserts
When a field is `undefined` in the JS insert object, `JSON.stringify` drops the key entirely. PostgREST then generates an INSERT without that column. For NOT NULL columns with no DEFAULT this should fail — but if the column is nullable or has an implicit default, it may silently insert NULL or an empty string. Route-level validation is more reliable than relying on DB constraints alone.

---

## Next.js

### params vs searchParams
- `params` — dynamic route segments (`[brandSlug]` → `{ brandSlug: "bikershades" }`). Only available when the folder has `[]` in its name.
- `searchParams` — query string (`?foo=bar`). Available as a prop on server pages.
- `useParams()` / `usePathname()` / `useSearchParams()` — client-side hooks, all synchronous (not Promises). The `params` prop on server components became a Promise in Next.js 15; these hooks did not.

### useParams instead of prop drilling
Sidebar used to receive `currentBrandSlug` as a prop from the layout. Switched to `useParams<{ brandSlug: string }>()` — reads directly from the URL, removes an unnecessary prop.

### usePathname
Returns the full path minus query string (`/admin/bikershades/orders`). Used in sidebar for active state detection. `startsWith` covers sub-routes; `===` used for the overview (exact match only).

### Server vs client components
- Server components run once on request — `await params`, fetch data, no hooks.
- Client components (`"use client"`) can't await params directly. Must use hooks or receive values as props. State and effects only exist on the client.
- Passing data server → client: server component fetches, passes as `initialX` prop to client component.

---

## React

### Why useState not const for submissions
A plain `const` resets to `initialSubmissions` on every render. `useState` persists across re-renders. Updating it via `setSubmissions` triggers a re-render and the new value sticks. The server component only runs on the initial request — to reflect a status update without a full page reload, client state is required.

### New reference for state updates
`Array.map` returns a new array reference. React uses reference equality to detect state changes. Mutating the existing array in place and calling `setState(sameRef)` may not trigger a re-render. However, since `setSaving(null)` runs in the same batch and causes a re-render anyway, the new reference from `map` is technically redundant for triggering the render — but immutability is still the correct React pattern.

### State is queued, not applied immediately
`setSaving(id)`, `setSubmissions(...)`, `setSaving(null)` don't apply mid-function. They're queued. React batches and flushes them after the async function. In React 18, all state updates are batched — including those after `await`.

### Pure functions outside components
If a function doesn't close over component state/props, define it outside. It's a pure function (same input → same output), not recreated on every render, and easier to test. Pass component values as parameters instead of closing over them (e.g. `statusColor(status, accent)` lives outside the component).

---

## Draft/Save Pattern

The status dropdown uses a draft-then-save UX:

```
draftStatus: Record<id, status>  — initialized from DB values
submissions: TbybSubmission[]    — source of truth for committed status
saving: string | null            — id of submission currently being saved
```

- Dropdown selection updates `draftStatus[id]` only
- Save button enabled when `draftStatus[id] !== sub.status`
- On save: update `submissions` so `sub.status` matches `draftStatus[id]` → button auto-disables
- `saving !== null` disables all save buttons; `saving === sub.id` shows "Saving…" on the active one only
- No cleanup needed: after save, draft equals committed status, so the diff check naturally disables the button
