# Updates Since Last LEARNINGS.md (July 1, 2026)

---

## Supabase `auth.getUser()` Destructuring

```ts
const { data: { user } } = await supabase.auth.getUser();
```

`data:` is the key to look up, `{ user }` is what to bind from its value. `data` is not available as a variable — only `user` is. Same as:

```ts
const { data } = await supabase.auth.getUser();
const { user } = data;
```

On an invalid token, Supabase sets `data.user` to `null` and `error` to set. The `if (!user) return null` check covers both — no need to destructure `error` separately.

---

## Why You Can't Verify Before Creating the Client

`auth.getUser()` is a method on the Supabase client — the client has to exist to call it. You can't verify a token without first instantiating the client. Both admin and user clients follow this pattern: create first, verify after.

---

## Admin Client vs User Client for User Queries

Using the admin client for user-scoped queries bypasses RLS — you'd have to manually add `.eq("user_id", user.id)` to every query. The user client passes the JWT in headers so RLS automatically scopes all queries to the logged-in user. No manual filtering needed.

---

## Webhook Responses

The webhook endpoint is called by Stripe's servers, not the browser. The client never sees the response — it just needs to be a 200 so Stripe knows the event was received.

---

## `priceChanged` Short-Circuit

```ts
priceChanged: dbPrice !== null && dbPrice !== item.priceCents
```

If `dbPrice` is `null` (item doesn't exist), the first condition fails and `priceChanged` is `false`. Reporting a price change when the item doesn't exist would be meaningless — `exists: false` already signals the problem.

---

## Variations Always Return an Array

Both `/products` and `/item` always return variations as an array — `[]` if none, never `null`. The `product.variations ?? []` fallback handles the case where Supabase returns null for an empty relation before `.map()` is called.

---

## Name and Image Are Always Paired

Image `name` and `src` come from the same DB row — if one exists, the other does too. No need to handle them independently or guard against one being present without the other.

---

## README Changes

- Response shape updated: `"error"` field renamed to `"message"` to match the current `err()` helper
- Third response shape added for validate-cart and checkout errors: `{ success: false, message, data: [] }`
- Tests section removed — no tests directory exists

