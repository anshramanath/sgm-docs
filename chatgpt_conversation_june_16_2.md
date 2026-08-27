# Next.js, Supabase, CORS, Context, and SEO Notes (June 16, 2026)

This document captures the main concepts discussed around building a separate frontend/backend storefront with Next.js, Supabase, server/client components, CORS, auth cookies, context providers, product grids, load more behavior, and SEO-friendly filtering.

---

## 1. CORS Mental Model

CORS stands for Cross-Origin Resource Sharing.

CORS is not really how a backend protects private data. It is how a backend tells browsers which frontend origins are allowed to read its responses.

Example origins:

```txt
Frontend: https://bikershades.vercel.app
Backend:  https://sunglass-monster-server.vercel.app
```

Even if both are part of the same project, they are different origins because the domains are different.

Browsers enforce CORS. Servers do not.

So this works without CORS problems:

```txt
Next.js server component -> backend API
```

Because it is server-to-server.

This can fail because of CORS:

```txt
Browser/client component -> backend API
```

Because the browser is making the request directly from one origin to another.

---

## 2. Why Server Components Do Not Need CORS

A Next.js server component runs on the Next.js server.

Flow:

```txt
Browser -> Next.js server -> backend API
```

The browser is not directly requesting the backend. The Next.js server is.

CORS is a browser rule, so the server can fetch from the backend without needing the backend to include CORS headers.

A client component runs in the user’s browser.

Flow:

```txt
Browser -> backend API
```

Now the browser checks whether the backend response includes headers that allow the frontend origin.

---

## 3. Access-Control-Allow-Origin

The main CORS header is:

```http
Access-Control-Allow-Origin: https://your-frontend.com
```

This means:

```txt
This backend allows browser JavaScript from this frontend origin to read this response.
```

Important: the value should be the frontend origin, not the backend origin.

Correct:

```http
Access-Control-Allow-Origin: https://bikershades.vercel.app
```

Not useful for allowing that frontend:

```http
Access-Control-Allow-Origin: https://sunglass-monster-server.vercel.app
```

You can also use:

```http
Access-Control-Allow-Origin: *
```

That means any frontend origin can read the response. This may be fine for public APIs, but it is not ideal for private or credentialed APIs.

---

## 4. Access-Control-Allow-Credentials

The credentials CORS header is:

```http
Access-Control-Allow-Credentials: true
```

This allows a browser-based frontend to send credentials with a cross-origin request.

Credentials usually means:

```txt
cookies
authorization headers
TLS client certificates
```

Most commonly, it means cookies.

Frontend request:

```ts
fetch("https://api.example.com/api/cart", {
  credentials: "include",
});
```

Backend response:

```http
Access-Control-Allow-Origin: https://your-frontend.com
Access-Control-Allow-Credentials: true
```

The browser reads this as:

```txt
This exact frontend origin is allowed to send credentials and read the response.
```

Important rule:

```http
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
```

is invalid. If credentials are allowed, the backend must name the exact frontend origin.

---

## 5. Can a Frontend Fake CORS?

No. A frontend cannot modify the backend response and add itself to CORS.

CORS headers are response headers. They must come from the backend.

This does not work:

```ts
fetch("https://backend.com/api/private", {
  headers: {
    "Access-Control-Allow-Origin": "https://evil-site.com",
  },
});
```

That is a request header, and the browser does not treat it as permission.

The real flow is:

```txt
Frontend JS makes request
        ↓
Browser adds Origin header
        ↓
Backend receives request
        ↓
Backend decides which CORS headers to return
        ↓
Browser checks response headers
        ↓
Browser either exposes or blocks the response body
```

A malicious site cannot make the browser pretend its origin is your frontend origin.

But someone using curl, Postman, or their own server can directly call your backend. That is why CORS is not full backend security.

CORS protects browser access. Auth protects data.

---

## 6. Is the Data Sent Back if CORS Fails?

Usually, yes.

For a simple request, the backend may still send the response, but the browser refuses to give the response body to the frontend JavaScript.

Flow:

```txt
Frontend JS makes request
        ↓
Backend sends response
        ↓
Browser receives response
        ↓
Browser sees no matching Access-Control-Allow-Origin
        ↓
Browser blocks JS from reading the data
```

So your code sees a CORS error instead of the data.

There is also a preflight case.

For certain requests, the browser sends an `OPTIONS` request first:

```txt
Browser: Is this frontend allowed to make this actual request?
```

If the preflight fails, the browser may never send the real request.

Summary:

```txt
Simple request:
real request may be sent, response comes back, browser blocks JS from reading it.

Preflighted request:
browser asks first, and if denied, the real request is not sent.
```

---

## 7. Why Server Is Often Better Than Client

Server-side work is better for:

```txt
secure data access
secrets/API keys
auth checks
SEO
initial page load
avoiding CORS
reducing browser JavaScript
```

Client-side work is better for:

```txt
cart state
dropdowns
modals
bookmarks
localStorage
button clicks
live UI interactions
animations
```

For the storefront:

```txt
Product list page       -> server-first
Product detail page     -> server-first
Category page           -> server-first
Add to cart button      -> client
Cart drawer             -> client
Checkout session create -> server action / route handler
Admin actions           -> server action with auth checks
```

Clean rule:

```txt
Use server components for protected, SEO-important, or data-heavy work.
Use client components for interactive UI and browser-only state.
```

---

## 8. Server Actions Are Like Endpoints

A file with:

```ts
"use server";
```

can export server actions that client components/forms can call.

Example:

```ts
"use server";

export async function deleteProduct(productId: string) {
  await db.product.delete({ where: { id: productId } });
}
```

Even though the implementation is hidden from the browser, the ability to call the action is exposed.

So this should be thought of like:

```txt
POST /hidden-next-server-action
body: { productId }
```

That means server actions still need:

```txt
authentication
authorization
input validation
ownership checks
admin checks
```

Bad:

```ts
"use server";

export async function deleteProduct(productId: string) {
  await db.product.delete({ where: { id: productId } });
}
```

Better:

```ts
"use server";

export async function deleteProduct(productId: string) {
  const user = await getCurrentUser();

  if (!user) {
    throw new Error("Not logged in");
  }

  const isAdmin = await checkIfAdmin(user.id);

  if (!isAdmin) {
    throw new Error("Unauthorized");
  }

  await db.product.delete({ where: { id: productId } });
}
```

Mental model:

```txt
"use server" hides the implementation and secrets.
Auth checks protect the action from bad calls.
```

---

## 9. Server Actions Can Use Secrets Safely

If a client component imports a server action, the actual secret-using code does not get bundled into the browser.

Example:

```ts
// app/actions.ts
"use server";

export async function createCheckoutSession(productId: string) {
  const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

  const session = await stripe.checkout.sessions.create({
    // checkout config
  });

  return session.url;
}
```

Client component:

```tsx
"use client";

import { createCheckoutSession } from "@/app/actions";

export function BuyButton({ productId }: { productId: string }) {
  async function handleClick() {
    const url = await createCheckoutSession(productId);
    window.location.href = url;
  }

  return <button onClick={handleClick}>Buy</button>;
}
```

The client gets a callable reference, not the actual secret-using implementation.

Flow:

```txt
Client button click
        ↓
Browser calls Next.js server action endpoint
        ↓
Server runs function
        ↓
Server uses secret key privately
        ↓
Server returns safe result
```

Still, never trust arguments from the client.

Do not trust:

```txt
price
userId
isAdmin
ownership
coupon/discount values
```

Always verify on the server.

---

## 10. Supabase Auth Storage: Browser Client vs SSR Client

There are two different ideas:

```ts
import { createClient } from "@supabase/supabase-js";
```

This is the normal Supabase JS client. In browser apps, it commonly stores auth session data in localStorage by default.

But your setup uses:

```ts
import { createBrowserClient } from "@supabase/ssr";

export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  );
}
```

This is the Supabase SSR browser client.

Even though it runs in the browser, it is designed to work with the SSR cookie-based auth setup.

So your actual setup is:

```txt
Browser client:
createBrowserClient from @supabase/ssr
        ↓
login happens in client/browser
        ↓
session is synced into cookies
        ↓
server client reads cookies
        ↓
server component can get the user
```

That is why you were able to log in on the client and then read the user email from a server component.

---

## 11. Server Supabase Client Reads Cookies for User Auth

For user-aware Supabase on the server, the server client needs to read the user session from the incoming request.

In Next.js, that usually means importing cookies:

```ts
import { createServerClient } from "@supabase/ssr";
import { cookies } from "next/headers";

export async function createClient() {
  const cookieStore = await cookies();

  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return cookieStore.getAll();
        },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value, options }) =>
            cookieStore.set(name, value, options)
          );
        },
      },
    }
  );
}
```

This means:

```txt
Create a Supabase client on the server,
but use this request's cookies as the auth storage.
```

Important nuance:

```txt
Server Supabase needs cookies when it needs to know the logged-in user.
```

A service role/admin client does not need cookies:

```ts
const supabaseAdmin = createClient(
  SUPABASE_URL,
  SUPABASE_SERVICE_ROLE_KEY
);
```

But the service role bypasses normal user auth/RLS, so it must be protected manually with server-side checks.

Mental model:

```txt
User-aware server client:
reads cookies -> knows current user -> respects user auth/RLS

Service role/admin client:
uses secret key -> does not need cookies -> must manually protect permissions
```

---

## 12. What Gets Stored in Cookies

For Supabase SSR auth, cookies mainly store session/auth material.

Usually this means:

```txt
access token / JWT
refresh token
expiration info
some session metadata
```

The cookie is not where your full app profile should live.

Do not store these as your main app data in cookies:

```txt
full profile
cart
orders
admin settings
store ownership data
```

Instead:

```txt
Cookies:
auth/session

Supabase auth.users:
identity/email

profiles/admins/store tables:
roles, permissions, display name, ownership

localStorage:
cart, bookmarks, theme, non-sensitive browser state
```

For security-sensitive server checks, prefer validating the user with Supabase Auth instead of blindly trusting raw cookie/session contents.

---

## 13. Server-Side Sign In With Supabase

Using a server action for sign in is a good pattern in Next.js + Supabase SSR.

Example:

```ts
// app/login/actions.ts
"use server";

import { createClient } from "@/lib/supabase/server";
import { redirect } from "next/navigation";

export async function signIn(formData: FormData) {
  const email = String(formData.get("email"));
  const password = String(formData.get("password"));

  const supabase = await createClient();

  const { error } = await supabase.auth.signInWithPassword({
    email,
    password,
  });

  if (error) {
    return { error: error.message };
  }

  redirect("/dashboard");
}
```

Login page:

```tsx
// app/login/page.tsx
import { signIn } from "./actions";

export default function LoginPage() {
  return (
    <form action={signIn}>
      <input name="email" type="email" />
      <input name="password" type="password" />
      <button type="submit">Sign in</button>
    </form>
  );
}
```

Flow:

```txt
login form submits
      ↓
server action runs
      ↓
Supabase server client signs user in
      ↓
auth cookies are set
      ↓
server components/actions can read the user later
```

Do not use the Supabase service role key for normal sign in.

Use the public Supabase URL and anon/publishable key for normal SSR auth.

---

## 14. Cookies vs localStorage

Cookies and localStorage are different browser storage systems.

```txt
cookies != localStorage
```

localStorage:

```txt
readable by browser JavaScript
manually sent if you attach it
useful for non-sensitive client state
```

Good for:

```txt
cart
bookmarks
theme
recently viewed items
```

Bad for sensitive auth tokens because JavaScript can read them.

Cookies:

```txt
stored separately by the browser
automatically sent to matching domains
can be HttpOnly so JavaScript cannot read them
better for auth/session flows
```

For Supabase SSR:

```txt
session cookies let server components/actions know who is logged in
```

For localStorage:

```txt
server components cannot read it
```

---

## 15. Context Provider Mental Model

A context provider shares state across components.

The key is state, not just a normal variable.

Example:

```tsx
const [cart, setCart] = useState([]);

return (
  <CartContext.Provider value={{ cart, setCart }}>
    {children}
  </CartContext.Provider>
);
```

Any component inside the provider can read:

```tsx
const { cart, setCart } = useCart();
```

When one component calls:

```tsx
setCart([...cart, newItem]);
```

React does this:

```txt
state changes in Provider
        ↓
Provider value changes
        ↓
components using that context re-render
        ↓
they see the new cart
```

A normal variable does not work the same way:

```tsx
let cart = [];

function addItem(item) {
  cart.push(item);
}
```

React does not automatically track normal variable changes.

The clean mental model:

```txt
Context shares the value.
State tracks changes.
setState triggers re-renders.
Components using the context receive the updated value.
```

---

## 16. useEffect Is Not Needed Just for Context Updates

You do not need `useEffect` just to make context update other components.

This is enough:

```tsx
const [cart, setCart] = useState([]);
```

When you call:

```tsx
setCart(newCart);
```

React automatically re-renders components using that context.

Use `useEffect` for side effects, not normal UI updating.

Good uses for `useEffect`:

```txt
save cart to localStorage
load cart from localStorage on first mount
listen for browser events
fetch data from the client
sync with outside systems
```

Example:

```tsx
useEffect(() => {
  localStorage.setItem("cart", JSON.stringify(cart));
}, [cart]);
```

This means:

```txt
Whenever cart changes, save it to localStorage.
```

But the UI updated because of `setCart`, not because of `useEffect`.

---

## 17. Client Providers Wrapping Server Children

A client component can wrap server-rendered children through composition.

Example:

```tsx
// app/layout.tsx - server component by default
import CartProvider from "@/components/CartProvider";

export default function Layout({ children }: { children: React.ReactNode }) {
  return (
    <CartProvider>
      {children}
    </CartProvider>
  );
}
```

Cart provider:

```tsx
// components/CartProvider.tsx
"use client";

export default function CartProvider({ children }: { children: React.ReactNode }) {
  return (
    <CartContext.Provider value={/* cart state */}>
      {children}
    </CartContext.Provider>
  );
}
```

What happens:

```txt
Server renders the page/server children first
        ↓
Client provider wraps the rendered output
        ↓
Browser receives HTML for server content
        ↓
Only the provider and other client pieces hydrate with JS
```

Bundling:

```txt
Client provider JS is sent to browser.
Server child component code is not sent to browser.
Rendered HTML/result of the server child is sent.
```

SEO:

```txt
Server child content can still appear in the initial HTML.
Wrapping it in a client provider does not automatically destroy SEO.
```

Important limitation:

```txt
Server components inside the provider cannot read client context.
```

This does not work in a server component:

```tsx
const cart = useCart();
```

Only client components can use client context.

---

## 18. Which Pages Matter for SEO

Not all pages matter equally for SEO.

SEO-important pages:

```txt
home page
category pages
product detail pages
brand/collection pages
content/blog pages
```

Examples:

```txt
/prescription-sunglasses
/motorcycle-sunglasses
/category/bifocals
/product/riding-glasses-black-frame
```

These should be server-first, with real content in the initial HTML.

Pages that usually do not matter much for SEO:

```txt
/cart
/checkout
/login
/account
/admin
/order-confirmation
/settings
```

These can be client-heavy because Google does not need to rank a user’s cart or admin dashboard.

Clean rule:

```txt
Public discovery pages = SEO matters = server-first.
Private/interactive utility pages = SEO does not matter much = client-heavy is fine.
```

---

## 19. Client Product Grid and SEO

A client component can still contribute to SEO if its meaningful content is included in the initial HTML.

Client component does not always mean:

```txt
empty until browser JavaScript runs
```

In Next.js, client components can be pre-rendered into initial HTML and then hydrated.

This can be SEO-visible:

```tsx
"use client";

export function ProductGrid({ initialProducts }) {
  return (
    <div>
      {initialProducts.map((product) => (
        <a href={`/product/${product.slug}`} key={product.id}>
          {product.name}
        </a>
      ))}
    </div>
  );
}
```

Because `initialProducts` already exists when the page renders.

Bad SEO pattern:

```tsx
"use client";

export function ProductGrid() {
  const [products, setProducts] = useState([]);

  useEffect(() => {
    fetch("/api/products").then(...);
  }, []);

  return <div>{products.map(...)}</div>;
}
```

Initial HTML is empty/loading-only. Products appear only after browser JS runs.

Good SEO pattern:

```txt
Server page fetches first product batch
        ↓
passes products into client component
        ↓
client component renders them immediately
        ↓
load more button fetches additional pages
```

The better question is not:

```txt
Is ProductGrid client or server?
```

The better question is:

```txt
Are product names, links, images, and category text in the initial HTML?
```

If yes, it can help SEO.

---

## 20. Fetch More / Load More Product Grid

If your `FetchMoreProducts` component is client-side only and starts empty, SEO is weaker.

Bad:

```tsx
"use client";

export function FetchMoreProducts() {
  const [products, setProducts] = useState([]);

  useEffect(() => {
    fetch("/api/products?page=1").then(...);
  }, []);

  return <ProductGrid products={products} />;
}
```

Initial HTML:

```html
<div>Loading...</div>
```

Better:

```tsx
// app/category/[slug]/page.tsx
import FetchMoreProducts from "./FetchMoreProducts";

export default async function CategoryPage({ params }) {
  const initialProducts = await getProducts({
    categorySlug: params.slug,
    page: 1,
  });

  return (
    <>
      <h1>Motorcycle Sunglasses</h1>
      <FetchMoreProducts
        initialProducts={initialProducts}
        categorySlug={params.slug}
      />
    </>
  );
}
```

Client fetch-more component:

```tsx
"use client";

export default function FetchMoreProducts({ initialProducts, categorySlug }) {
  const [products, setProducts] = useState(initialProducts);
  const [page, setPage] = useState(1);

  async function loadMore() {
    const nextPage = page + 1;

    const res = await fetch(`/api/products?category=${categorySlug}&page=${nextPage}`);
    const moreProducts = await res.json();

    setProducts([...products, ...moreProducts]);
    setPage(nextPage);
  }

  return (
    <>
      <ProductGrid products={products} />
      <button onClick={loadMore}>Load more</button>
    </>
  );
}
```

Best pattern:

```txt
/category/sunglasses
-> server renders page 1 products

Load More button
-> client fetches page 2+

Optional crawlable URLs:
/category/sunglasses?page=2
/category/sunglasses?page=3
```

This gives users a nice load-more experience while keeping SEO strong.

---

## 21. Why Put Filters in the URL

Using URL params makes filtered views real, addressable page states.

Example:

```txt
/category/sunglasses?color=black&style=biker&inStock=true
```

Instead of hidden React state:

```txt
color = black
style = biker
inStock = true
```

URL params are useful because they give you:

```txt
shareable links
browser back/forward support
refresh keeps filters
server components can fetch filtered products
Google can discover useful filtered pages
users can land directly on filtered results
```

If filters only live in client state + useEffect:

```txt
/category/sunglasses
        ↓
client state says color=black
        ↓
useEffect fetches filtered products
```

The initial HTML is usually the generic category page, not the filtered page.

With URL params:

```tsx
export default async function CategoryPage({ searchParams }) {
  const color = searchParams.color;

  const products = await getProducts({ color });

  return <ProductGrid products={products} />;
}
```

The filtered products are rendered into the initial HTML.

---

## 22. Should Every Filter Combination Be Indexed?

No.

Some filtered pages are valuable for SEO:

```txt
/motorcycle-sunglasses
/prescription-sunglasses
/category/sunglasses?lens=polarized
/category/sunglasses?color=black
/category/bifocals?strength=2.00
```

Those match things people might search.

But random combinations should usually not all be indexed:

```txt
/category/sunglasses?color=black&sort=price_desc&page=7&inStock=true&brand=x&size=medium
```

That can create too many thin or duplicate pages.

Clean strategy:

```txt
Important category/filter pages:
use real URLs, server render, indexable

Random UI filters/sorts:
use URL params for usability, but canonical/noindex carefully
```

SEO-useful filters may include:

```txt
lens type
product type
bifocal strength
color
use case, like motorcycle/riding/prescription
```

Usually not SEO-useful:

```txt
sort=price_asc
inStock=true
page=7
very specific multi-filter combinations
```

---

## 23. Recommended Storefront Architecture

For your storefront, a clean architecture is:

```txt
Server-rendered public pages:
- home
- category pages
- product detail pages
- search pages where useful
- important filtered collection pages

Client components:
- cart provider
- cart drawer
- add-to-cart button
- bookmark button
- dropdown menus
- load more button
- filter UI controls

Server actions / route handlers:
- sign in
- sign out
- checkout session creation
- admin product updates
- protected database writes
```

For product grids:

```txt
Server fetches first product page.
Client Load More fetches additional pages.
URL params represent filters.
Important filter URLs can be SEO pages.
```

For auth:

```txt
Use @supabase/ssr.
Use createBrowserClient in browser code.
Use createServerClient with cookies in server code.
Use server actions for sign in if desired.
Do not use service role key for normal user login.
Use auth checks inside server actions.
```

For CORS:

```txt
Server components avoid browser CORS because requests are server-to-server.
Client components need backend CORS headers for cross-origin requests.
CORS is not real data security.
Backend auth and authorization are still required.
```

---

## 24. Final Mental Models

### CORS

```txt
Access-Control-Allow-Origin
= which frontend origin can read the backend response in the browser

Access-Control-Allow-Credentials
= whether that frontend can send cookies/auth credentials cross-origin

Auth
= whether the user/request is actually allowed to access data
```

### Server vs Client

```txt
Server component
= private, SEO-friendly, no browser CORS, good for data fetching

Client component
= interactive, browser state, event handlers, localStorage, UI actions
```

### Supabase SSR Auth

```txt
createBrowserClient from @supabase/ssr
= browser-side part of cookie-based SSR auth

createServerClient from @supabase/ssr
= server-side client that reads auth from cookies

service role client
= backend/admin power, no cookies needed, manually protect it
```

### Context

```txt
Context shares state.
useState tracks changes.
setState causes re-render.
useEffect is only for side effects.
```

### SEO

```txt
The key SEO question is not simply whether a component is client or server.
The key question is whether important content appears in the initial HTML.
```

### Filters

```txt
React state filters
= temporary UI state

URL param filters
= real page state, shareable, refresh-safe, server-renderable, potentially SEO-friendly
```

