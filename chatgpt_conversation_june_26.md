# React Effects, Next.js Server Helpers, Supabase SSR/Browser Clients, Cookies, and Auth Network Flow Notes (June 26, 2026)

## 1. `useEffect` cleanup basics

A `useEffect` only has cleanup behavior if it returns a function.

```tsx
useEffect(() => {
  console.log("mounted");

  return () => {
    console.log("cleanup");
  };
}, []);
```

With an empty dependency array:

```txt
effect body:
  runs once on mount

returned cleanup:
  runs on unmount
```

If there is no return:

```tsx
useEffect(() => {
  console.log("mounted");
}, []);
```

then nothing special runs on unmount.

---

## 2. Cleanup does not only mean unmount

The cleanup function runs in two situations:

```txt
1. when the component unmounts
2. before the effect reruns because dependencies changed
```

Example:

```tsx
useEffect(() => {
  console.log("effect ran for", productId);

  return () => {
    console.log("cleanup for", productId);
  };
}, [productId]);
```

If `productId` changes from `1` to `2`, React does:

```txt
cleanup for 1
effect ran for 2
```

Then when the component unmounts:

```txt
cleanup for 2
```

### Clean rule

```txt
No return:
  no cleanup

Return function:
  cleanup runs before rerun and on unmount

Return function + []:
  cleanup runs only on unmount
```

---

# Next.js `"use server"` and Network Requests

## 3. What `"use server"` means

`"use server"` means the code is server-only / server-action capable.

It controls **where code runs**.

It does not automatically mean a network request happens just because the file exists.

The network behavior depends on **who calls the function**.

---

## 4. Server Component calling server code

If a Server Component calls a normal server helper during rendering:

```tsx
// app/products/page.tsx
const products = await getProducts();
```

and `getProducts()` runs on the server:

```ts
export async function getProducts() {
  const supabase = await createClient();
  return supabase.from("products").select("*");
}
```

Flow:

```txt
Browser requests page
↓
Next server renders page
↓
getProducts runs on server
↓
Next server calls Supabase
↓
page response returns
```

There is no extra browser → server request beyond the original page request.

There is still a server → Supabase network request when the query runs.

---

## 5. Client Component calling an API route

If a Client Component does:

```tsx
"use client";

useEffect(() => {
  fetch("/api/products");
}, []);
```

Flow:

```txt
Browser loads page
↓
Client JS runs
↓
Browser makes extra request to /api/products
↓
API route runs on Next server
↓
Next server calls Supabase
```

That creates an extra browser → Next server request.

---

## 6. Client Component calling a server action

If a Client Component calls a server action:

```ts
"use server";

export async function updateCart(productId: string) {
  // server code
}
```

from:

```tsx
"use client";

await updateCart(productId);
```

Flow:

```txt
Browser
↓
Next server action request
↓
server action runs
↓
maybe server calls Supabase/backend
```

That is also an extra browser → Next server request.

---

## 7. Client Component using Supabase browser client directly

If the Client Component uses the Supabase browser client:

```tsx
"use client";

const supabase = createBrowserClient(...);

const { data } = await supabase
  .from("products")
  .select("*");
```

Flow:

```txt
Browser
↓
Supabase directly
```

There is no extra request to your Next server, but there is a browser → Supabase network request.

---

## 8. Core network mental model

```txt
"use server":
  controls where code executes

Supabase query:
  causes request to Supabase

Client calling server action:
  causes browser → Next server request

Client calling API route:
  causes browser → Next server request

Server component calling server helper:
  no extra browser request beyond original page render
```

---

# Supabase Server Client vs Browser Client

## 9. Supabase server client with SSR cookies

A server Supabase client often looks like:

```ts
import { cookies } from "next/headers";
import { createServerClient } from "@supabase/ssr";
```

This is server-only.

It can be used in:

```txt
Server Components
Route Handlers
Server Actions
server-side helpers
```

It cannot be used in:

```txt
Client Components
browser event handlers
"use client" files
```

because `next/headers` only exists on the Next.js server.

---

## 10. Why the server client needs `cookies()`

Server code is not literally inside the browser.

So it cannot directly access browser storage.

Instead, when the browser requests a page/API route, it sends cookies with that request.

Then the Next server can read those cookies:

```ts
const cookieStore = await cookies();
```

Flow:

```txt
Browser sends request with cookies
↓
Next server receives request
↓
server code reads cookies using next/headers
```

Reading cookies this way does not call Supabase by itself.

It just reads the incoming request cookies on the Next server.

---

## 11. Does creating the Supabase server client cause a network request?

Usually no.

This:

```ts
const supabase = createServerClient(...);
```

or:

```ts
const supabase = await createClient();
```

mostly creates a configured client object.

This does not query Supabase by itself.

The network request happens when you do something like:

```ts
await supabase.from("products").select("*");
```

or some auth methods that verify with Supabase.

---

## 12. Supabase auth methods: `getSession()` vs `getUser()`

### `getSession()`

```ts
const { data } = await supabase.auth.getSession();
```

This generally reads the current session/token from storage/cookies.

Mental model:

```txt
getSession:
  read the session you already have
```

It may not fully validate the token with Supabase every time.

### `getUser()`

```ts
const { data } = await supabase.auth.getUser();
```

This generally verifies the token with Supabase Auth.

Mental model:

```txt
getUser:
  ask Supabase Auth who this token belongs to
```

For protected server routes, `getUser()` is safer because it validates the token.

---

## 13. Supabase browser client

The browser client is for client-side React code.

Example:

```ts
import { createBrowserClient } from "@supabase/ssr";

export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  );
}
```

Use it in:

```tsx
"use client";

const supabase = createClient();
```

The browser client can interact with browser-accessible auth storage/cookies because it runs inside the browser.

---

## 14. Can the browser client use `next/headers cookies()`?

No.

This cannot be used in browser/client code:

```ts
import { cookies } from "next/headers";
```

`cookies()` from `next/headers` is only for the Next server.

### Clean rule

```txt
createServerClient + next/headers cookies:
  server only

createBrowserClient:
  browser/client components

Anon key:
  okay in browser

Service role key:
  server only, never browser
```

---

# Cookies, HttpOnly, and CSRF

## 15. What is `HttpOnly`?

`HttpOnly` is a cookie setting that means:

```txt
JavaScript cannot read this cookie.
```

Example:

```http
Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Lax
```

Frontend JS cannot access this cookie through:

```js
document.cookie
```

But the browser can still automatically send it with requests.

---

## 16. HttpOnly mental model

```txt
HttpOnly cookie:
  browser can store it
  browser can send it
  server can read it
  JavaScript cannot read it
```

So a browser Supabase client cannot directly read an HttpOnly cookie.

But if the browser makes a request to the matching domain, the browser network layer can still attach that cookie automatically.

---

## 17. Why HttpOnly helps

HttpOnly helps protect sensitive cookies/tokens from XSS token theft.

Without HttpOnly:

```txt
malicious JS runs on your site
↓
reads token from localStorage/cookie
↓
sends token to attacker
```

With HttpOnly:

```txt
malicious JS runs on your site
↓
tries to read cookie
↓
cannot access HttpOnly cookie directly
```

---

## 18. What is CSRF?

CSRF means Cross-Site Request Forgery.

It is a risk when your auth is based on cookies because the browser automatically sends cookies with requests.

Attack idea:

```txt
You are logged into your admin site
↓
Your browser has an auth cookie
↓
You visit evil-site.com
↓
evil-site tricks your browser into POSTing to your-site.com
↓
browser automatically includes your cookie
↓
your backend might think the request is legitimate
```

The attacker does not steal your cookie. They trick your browser into using it.

---

## 19. HttpOnly helps XSS, not CSRF by itself

```txt
HttpOnly:
  JS cannot read the cookie

But:
  browser still sends the cookie automatically
```

That automatic sending is why CSRF is possible.

---

## 20. CSRF protections

### SameSite cookies

Set cookies with:

```http
SameSite=Lax
```

or:

```http
SameSite=Strict
```

Rules:

```txt
SameSite=Strict:
  strongest, but can be annoying for login/link flows

SameSite=Lax:
  good default for many apps

SameSite=None:
  allows cross-site cookies, requires Secure, needs extra care
```

### CSRF tokens

Server gives the frontend a random token.

Frontend sends it back on dangerous requests:

```http
X-CSRF-Token: random_token_here
```

Server checks whether the token matches.

A malicious external site cannot easily know that token.

### Origin / Referer checks

Backend checks:

```http
Origin: https://your-site.com
```

If the request came from:

```txt
https://evil-site.com
```

reject it.

---

## 21. Bearer tokens vs cookie auth

If you use:

```http
Authorization: Bearer <token>
```

CSRF is less of a concern because browsers do not automatically attach custom Authorization headers for random cross-site forms.

If you use cookie auth:

```txt
browser automatically sends auth cookie
```

then CSRF matters more.

### Mental model

```txt
XSS:
  attacker tries to steal/use tokens through malicious JS

CSRF:
  attacker cannot read cookie, but tricks browser into sending it

HttpOnly:
  helps against XSS token theft

SameSite / CSRF token / Origin checks:
  help against CSRF
```

---

# JWT Access Tokens

## 22. JWT after login

After login, auth systems usually provide:

```txt
access token:
  short-lived token proving the user is logged in

refresh token:
  longer-lived token used to get new access tokens
```

The access token is often a JWT.

It may be valid for a short time, commonly around one hour depending on auth settings.

---

## 23. Can a JWT be stolen from the browser?

Yes, if it is accessible to frontend JavaScript and malicious JavaScript runs on your site.

Common danger:

```txt
token in localStorage/sessionStorage/browser-readable cookie
↓
XSS occurs
↓
attacker JS reads token
↓
attacker uses token until it expires
```

Short-lived access tokens reduce the damage window.

Refresh tokens are more sensitive and should be protected carefully.

---

## 24. Browser-readable session vs HttpOnly session

If the access token is browser-readable, client code can do:

```ts
const {
  data: { session },
} = await supabase.auth.getSession();

const token = session?.access_token;
```

Then the client can attach it:

```ts
headers: {
  Authorization: `Bearer ${token}`
}
```

If the auth cookie/token is HttpOnly, client JS cannot read it.

In that case, server code must read the cookie and attach/verify auth.

---

# Avoiding Extra Network Requests by Passing Tokens

## 25. Original issue

You had a TS file marked:

```ts
"use server";
```

because inside the file you used the Supabase server client to get the JWT/session from cookies.

Example shape:

```ts
"use server";

import { createClient } from "@/lib/supabase/server";

export async function authedFetch(path: string) {
  const supabase = await createClient();
  const { data } = await supabase.auth.getSession();

  const token = data.session?.access_token;

  return fetch(`${BASE}${path}`, {
    headers: {
      Authorization: `Bearer ${token}`,
    },
  });
}
```

If a Client Component calls this server function/server action, it creates:

```txt
Browser
↓
Next server action request
↓
server reads session/token
↓
server calls backend API
```

That adds an extra browser → Next server hop.

---

## 26. Better split: token-agnostic fetch helper

If the only server-only part is getting the token from cookies, separate that part.

Make the fetch helper accept the token as an argument:

```ts
export async function authedFetch<T>(
  path: string,
  token: string,
  options?: RequestInit
): Promise<T> {
  const res = await fetch(`${process.env.NEXT_PUBLIC_API_BASE_URL}${path}`, {
    ...options,
    headers: {
      ...options?.headers,
      Authorization: `Bearer ${token}`,
    },
  });

  if (!res.ok) {
    throw new Error(`Request failed: ${res.status}`);
  }

  return res.json();
}
```

This file does not need `"use server"` because it does not import server-only APIs like `cookies()`.

It just receives a token and performs a fetch.

---

## 27. Client usage

In a Client Component:

```tsx
"use client";

const supabase = createBrowserClient(...);

const {
  data: { session },
} = await supabase.auth.getSession();

if (!session) {
  throw new Error("Not logged in");
}

await authedFetch("/api/admin/products", session.access_token);
```

Flow:

```txt
Browser
↓
Backend API
```

instead of:

```txt
Browser
↓
Next server action
↓
Backend API
```

So yes, passing the token can avoid the extra Next server-action network hop.

---

## 28. Server usage

On the server, you can still have a server-only wrapper:

```ts
import { createClient } from "@/lib/supabase/server";
import { authedFetch } from "@/lib/api/authedFetch";

export async function serverAuthedFetch<T>(path: string): Promise<T> {
  const supabase = await createClient();

  const {
    data: { session },
  } = await supabase.auth.getSession();

  if (!session) {
    throw new Error("Not logged in");
  }

  return authedFetch<T>(path, session.access_token);
}
```

This wrapper can be server-only because it reads the session from cookies.

The lower-level `authedFetch(path, token)` stays reusable.

---

## 29. Best architecture

Use one low-level helper:

```txt
authedFetch(path, token)
```

Then create separate token sources:

```txt
Client side:
  get token from Supabase browser client
  pass token into authedFetch

Server side:
  get token from Supabase server client/cookies
  pass token into authedFetch
```

This avoids forcing the entire fetch helper to be server-only.

---

## 30. Important caveat

This works only if the client can read the token/session.

If your auth is HttpOnly-cookie-only, the browser client cannot read the token.

Then you need server-side code to read the cookie and attach auth.

### Rule

```txt
Browser-readable token:
  client can pass token to shared helper

HttpOnly token:
  client cannot read token
  server must handle auth
```

---

# Final Mental Model

## `use server`

```txt
"use server":
  code runs on server / can be a server action

It does not automatically cause a network request.

Network request happens when browser/client calls server code.
```

## Supabase clients

```txt
Supabase server client:
  uses next/headers cookies
  server only
  reads cookies from incoming request

Supabase browser client:
  runs in browser
  uses browser-accessible auth storage/cookies
```

## Network requests

```txt
Server Component → server helper:
  no extra browser request

Client Component → API route:
  extra browser → server request

Client Component → server action:
  extra browser → server request

Client Component → Supabase browser client:
  browser → Supabase request

Server helper → Supabase query:
  server → Supabase request
```

## Cookies/security

```txt
HttpOnly:
  JS cannot read cookie

Browser:
  can still send HttpOnly cookie automatically

Server:
  can read cookie from request

CSRF:
  risk when browser automatically sends cookies

Bearer token:
  less CSRF risk, but token theft matters if JS-readable
```

## Best code organization

```txt
Do not make a whole helper "use server" just to get a token.

Separate:
  1. token retrieval
  2. actual fetch logic

Pass the token into the fetch helper.

Use server token retrieval on the server.
Use browser token retrieval in client components.
```
