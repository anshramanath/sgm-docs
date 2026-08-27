# Session Learnings (July 2, 2026)

## getFiller Endpoint

Added `getFiller(n: number)` to `api.ts` — fetches N random products from `/api/public/filler`. Used in two places:

- **Homepage BestSellers**: replaces old logic that iterated categories sequentially until finding one with ≥10 products. Now just one call.
- **Product page Related**: replaces category-filtered related products. No need to pass `slug` or `categoryId` — occasional same-product collision is acceptable.

`getFiller` is lowercase so it can't be a JSX component directly. The pattern is: `getFiller` is the data helper, `BestSellers`/`Related` are the server components that call it.

---

## Public Image Organization

Images moved from flat root paths (`/bikershades-hero.jpg`) into per-brand subfolders (`/bikershades/hero.jpg`, `/bikershades/cat-1.jpg`, etc.). Each brand folder contains: `hero.jpg`, `logo.jpg`, `cat-1` through `cat-5`, `edit-1` and `edit-2`. `brand.ts` updated to match.

---

## Homepage Editorial Index

Category grid uses leaves `[0–4]`. Editorial split should pick up at index 5 — `leaves[(5 + i) % leaves.length]`, not `4 + i`.
