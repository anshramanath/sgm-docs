# Learnings & Build Notes (June 25, 2026)

## Error Pages

Three error pages, all consistent in format (centered, `mb-[20vh]`, large status code, heading, body, CTA button).

| Page | File | Trigger | CTA |
|------|------|---------|-----|
| 404 | `src/app/not-found.tsx` | Next.js `notFound()` or invalid route | Back to Home |
| 500 | `src/app/try-again/page.tsx` | Redirect from API failure | Back to Home |
| Error boundary | `src/app/error.tsx` | Unhandled React render exception | Try Again (calls `reset()`) |

**Key constraints:**
- `redirect()` cannot be called inside `not-found.tsx` — it throws a performance timing error. Full content lives directly in that file.
- `error.tsx` must be `"use client"` — it's a React error boundary that receives `reset()` to retry the same page.
- `error.tsx` uses `reset()` (retries the current page) not `href="/"` — clicking Try Again re-renders the failing route, not the homepage.
- `redirect()` returns `never` internally — TypeScript narrows the type after it, so no return is needed after a redirect call.

---

## brand.ts — Server-Only Pattern

```ts
"use server";

const BRANDS = { ... } as const; // not exported — stays private

export async function getBrand() {
  return BRANDS[process.env.BRAND_SLUG as keyof typeof BRANDS];
}
```

**Why `"use server"` not `"server-only"`:**
- `"server-only"` = hard build error if client imports the file
- `"use server"` = marks exported functions as server actions callable via RPC; file runs server-side only

**Why async:**
- `"use server"` requires exported functions to be `async`
- Callers must `await getBrand()` — `getBrand().slug` won't work since it returns a Promise

**`BRAND_SLUG` env var:**
- No `NEXT_PUBLIC_` prefix — server-side only, never in the client bundle
- Accessed via `process.env.BRAND_SLUG` inside server functions

---

## api.ts — Architecture

Three fetch helpers, each handles network failure by returning a synthetic `Response`:

| Helper | Use case | Auth |
|--------|---------|------|
| `apiFetch` | GET requests with query params | None |
| `publicPostFetch` | POST with JSON body, no auth | None |
| `authedFetch` | POST/PUT with JWT | Supabase Bearer token |

All synthetic error responses use `message` (not `error`) to match `ApiResponse` shape.

**Error handling pattern for public endpoints:**
```ts
export async function getX(): Promise<X> {
  const res = await apiFetch("/api/public/x", { brandSlug: BRAND_SLUG });
  const json: ApiResponse<X> = await res.json();

  if (!json.success) {
    switch (res.status) {
      case 404: notFound();       // item endpoints only
      case 500: redirect("/try-again");
      default:  redirect("/try-again");
    }
  }

  return json.data;
}
```

**Why the cast / narrowing works:**
- All branches in `if (!json.success)` call `redirect()` or `notFound()` — both return `never`
- TypeScript narrows `json` to the success branch after the if block
- No cast needed when the switch has a `default` that always throws

**`validateCart` is different:**
- Returns `{ data: ValidateCartItem[], status: number }` so the caller knows whether it was 200/404/409/422
- Only redirects on 500 — 404/409/422 carry per-item validation data the UI needs
- Uses `?? []` because `data?: E` in `ApiResponse` is optional even when E is specified

---

## TypeScript Lessons

**Narrowing through compound conditions:**
`if (!json.success && res.status === 500) redirect(...)` — after this, TypeScript can't conclude `json.success` is true because the condition could be false for two reasons (success is true, OR status isn't 500). Use separate checks or a cast.

**`async` functions always return `Promise<T>`:**
Even if the body is synchronous. Every async function must have a return path for every code branch, or TypeScript errors on missing return.

**`Array.isArray` narrows to `any[]`**, not a specific type — not useful as a TypeScript guard for typed arrays.

**`Response` bodies are streams** — can only be consumed once. Can't store and reuse a `Response` instance; each call needs `new Response(...)`.

**`notFound()` returns `never`** — fallthrough in a switch after `notFound()` is safe; TypeScript flags a `break` after it as unreachable.

**`E = never` in generics:**
`data?: never` means the field can technically exist but its value must be `never` — impossible in practice. Passing a real type as E (e.g. `ValidateCartItem[]`) makes `data?: ValidateCartItem[]` — optional, not required. The `?` is about presence, the generic is about type.

---

## Next.js Lessons

**`generateMetadata()`** — async alternative to static `metadata` export for layouts/pages. Required when metadata depends on async data (e.g. `getBrand()`).

**Server components can be non-async** — no error until you add `await`. Client components cannot be async.

**`layout.tsx` is innately a server component** — no `"use server"` needed, but `async` must be added explicitly if using `await`.

**Next.js deduplicates server action calls** — calling `getBrand()` multiple times in the same render pass is safe; it's only fetched once.

**HTTP status reference:**
| Code | Meaning |
|------|---------|
| 200 | Success |
| 400 | Bad request — missing/invalid params |
| 401 | Unauthorized |
| 404 | Not found |
| 409 | Conflict (e.g. price changed) |
| 422 | Unprocessable (multiple validation failures) |
| 500 | Server/DB error |
| 503 | Network failure (synthetic — never from the actual server) |
