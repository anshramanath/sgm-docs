# Next.js, Supabase, CORS, SEO, Legal, and React Notes (June 17, 2026)

This document captures the main things learned and planned during the recent discussion around building a production-ready ecommerce storefront with Next.js, Supabase, Stripe, React Context, SEO-friendly product pages, and basic website/legal safety.

---

## 1. CORS Mental Model

CORS stands for **Cross-Origin Resource Sharing**.

It is not really how a server protects private data. It is how a server tells **browsers** which frontend origins are allowed to read its responses.

Example:

```txt
Frontend:
https://bikershades.vercel.app

Backend:
https://api.bikershades.com
```

These are different origins. If browser JavaScript from the frontend tries to fetch the backend, the browser checks whether the backend response allows that frontend origin.

The main header is:

```http
Access-Control-Allow-Origin: https://bikershades.vercel.app
```

That means:

```txt
This backend allows browser JavaScript from this frontend origin to read the response.
```

Important distinction:

```txt
CORS does not stop the backend from sending a response.
CORS stops browser JavaScript from reading the response.
```

Tools like `curl`, Postman, bots, or another backend server can still call the API directly. That is why CORS is not enough for real security.

Real security needs:

```txt
authentication
authorization
server-side permission checks
API keys when appropriate
sessions/JWTs
RLS or database policies
input validation
```

---

## 2. Why Server Components Avoid CORS

In Next.js, server components and client components run in different places.

A server component fetch looks like:

```txt
Browser
  ↓
Next.js server
  ↓
Backend/API/database
```

That is a server-to-server request. Browsers enforce CORS. Servers do not.

A client component fetch looks like:

```txt
Browser
  ↓
Backend/API
```

That request comes directly from browser JavaScript, so CORS applies.

So this usually avoids CORS:

```ts
// server component
const res = await fetch("https://api.example.com/products");
```

But this may hit CORS:

```tsx
"use client";

useEffect(() => {
  fetch("https://api.example.com/products");
}, []);
```

Clean mental model:

```txt
Server component = no browser CORS because it runs on the server.
Client component = browser CORS applies because it runs in the browser.
```

---

## 3. Access-Control-Allow-Credentials

This header matters when cross-origin requests include credentials such as:

```txt
cookies
authorization headers
TLS client certificates
```

Most commonly, this is about cookies.

Backend response:

```http
Access-Control-Allow-Origin: https://your-frontend.com
Access-Control-Allow-Credentials: true
```

Frontend fetch:

```ts
fetch("https://your-backend.com/api/cart", {
  credentials: "include",
});
```

Meaning:

```txt
The frontend is asking the browser to include cookies/auth.
The backend is saying this exact frontend origin is allowed to do that.
```

Important rule:

```http
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
```

is not allowed. If credentials are included, the backend must name the exact allowed origin.

Mental model:

```txt
Access-Control-Allow-Origin
= which frontend can read the response

Access-Control-Allow-Credentials
= whether that frontend can include cookies/auth credentials
```

CORS does not prove the user is logged in. It only allows the browser to send the credentials. The backend still has to validate them.

---

## 4. Can a Frontend Fake CORS?

No, not from normal browser JavaScript.

A malicious frontend cannot add this to its request and make it work:

```http
Access-Control-Allow-Origin: https://evil-site.com
```

That is a response header. The browser only trusts it if it comes back from the backend.

The browser sends an `Origin` header:

```http
Origin: https://some-frontend.com
```

Then the backend decides whether to allow it.

Flow:

```txt
Frontend JS makes request
  ↓
Browser sends Origin header
  ↓
Backend responds with or without CORS headers
  ↓
Browser checks response headers
  ↓
Browser either exposes or blocks the response body
```

However, attackers can still call your backend outside the browser using Postman, curl, or their own server. That is why CORS is not backend security.

---

## 5. Server Actions Are Basically Endpoints

A `"use server"` function imported into a client component does not expose its implementation or secrets to the browser.

Example:

```ts
"use server";

export async function createCheckoutSession(productId: string) {
  // Uses Stripe secret key on server only
}
```

A client component can call it:

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

The client does not receive the real function body. It receives a callable reference. The real work runs on the Next.js server.

But this means server actions should be treated like endpoints:

```txt
Client component calls server action
  ↓
Next.js sends request to hidden server action endpoint
  ↓
Server runs the function
```

Therefore, every server action needs:

```txt
auth checks
authorization checks
input validation
business rule validation
error handling
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

  await db.product.delete({
    where: { id: productId },
  });
}
```

Clean mental model:

```txt
"use server" hides the implementation.
Auth and validation protect the action.
```

---

## 6. Supabase Browser Client vs SSR Client

There are different Supabase client patterns.

Plain browser Supabase client:

```ts
import { createClient } from "@supabase/supabase-js";
```

This is commonly browser/localStorage based for auth sessions.

Supabase SSR browser client:

```ts
import { createBrowserClient } from "@supabase/ssr";

export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  );
}
```

This is different. It is the browser-side part of Supabase’s SSR setup and is designed to work with cookie-based auth.

Server Supabase client:

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
          cookiesToSet.forEach(({ name, value, options }) => {
            cookieStore.set(name, value, options);
          });
        },
      },
    }
  );
}
```

This server client reads the auth session from cookies.

---

## 7. Why Server Supabase Reads Cookies

Server code cannot read browser `localStorage`.

So if you want this to work in a server component or server action:

```ts
const {
  data: { user },
} = await supabase.auth.getUser();
```

the server needs the user session from the incoming request.

In the Supabase SSR setup, that means cookies.

Clean mental model:

```txt
Browser localStorage
= only available to browser JavaScript

Cookies
= sent with HTTP requests
= readable by server code through the request
```

So this works:

```txt
Supabase SSR browser client signs in
  ↓
session gets stored/synced through cookies
  ↓
Next.js server component receives request with cookies
  ↓
Supabase server client reads cookies
  ↓
getUser() sees the logged-in user
```

---

## 8. What Gets Stored in Supabase Auth Cookies?

Mostly session data:

```txt
access token / JWT
refresh token
expiration info
some user/session metadata
```

The cookie is not where you should store full app profile data.

Better split:

```txt
Cookies:
auth session

Supabase auth.users:
identity/email/auth user record

profiles/admins/store tables:
roles, permissions, display name, ownership, app-specific data

localStorage:
cart, bookmarks, theme, non-sensitive browser state
```

For secure server decisions, prefer:

```ts
await supabase.auth.getUser();
```

over blindly trusting locally stored session values.

---

## 9. Server-Side Sign In with Supabase

A good plan is to use a server action for sign in.

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

Form:

```tsx
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
Login form submits
  ↓
server action runs
  ↓
Supabase server client signs user in
  ↓
auth cookies are set
  ↓
future server components/actions can read user
```

Do not use the Supabase service role key for normal sign in. Use the public URL and anon/publishable key with the SSR server client.

---

## 10. Avoiding Browser Supabase Entirely

A platform can mostly avoid using Supabase directly in the browser.

Architecture:

```txt
Browser/client components
  ↓
Next.js server actions / route handlers
  ↓
Supabase server client / database
```

The browser calls your functions:

```ts
await signIn(formData);
await addToCart(productId);
await updateProfile(formData);
await createCheckoutSession(cart);
```

Your server functions call Supabase.

Benefits:

```txt
more control
fewer direct browser-to-Supabase calls
secrets stay server-side
easier to centralize validation
better for server components
```

Tradeoff:

```txt
more backend code to write
less direct realtime/client convenience
you must build your own server actions/routes carefully
```

This is a good fit for a serious ecommerce storefront.

---

## 11. Context Provider Mental Model

A context provider shares state across components.

Example:

```tsx
const [cart, setCart] = useState([]);

return (
  <CartContext.Provider value={{ cart, setCart }}>
    {children}
  </CartContext.Provider>
);
```

Any child client component can read it:

```tsx
const { cart, setCart } = useCart();
```

When one component calls:

```tsx
setCart([...cart, newItem]);
```

React updates the provider state and re-renders components that use that context.

Flow:

```txt
Component A calls setCart()
  ↓
Provider state changes
  ↓
Provider value changes
  ↓
Components using context re-render
  ↓
Component B sees updated cart
```

Important:

```txt
Context shares the value.
State tracks changes.
setState triggers re-renders.
useEffect is not needed just to update other components.
```

---

## 12. When useEffect Is Needed

`useEffect` is not needed for normal context updates.

You need `useEffect` for side effects, such as:

```txt
saving cart to localStorage
loading cart from localStorage on mount
listening for browser events
fetching data from the client
syncing with an external system
setting timers
```

Example:

```tsx
useEffect(() => {
  localStorage.setItem("cart", JSON.stringify(cart));
}, [cart]);
```

That means:

```txt
Whenever cart changes, save it outside React.
```

The UI update itself happens because of `setCart`, not because of `useEffect`.

---

## 13. useRef Mental Model

`useRef` is helpful when you need to remember something without causing a re-render.

```tsx
const myRef = useRef(initialValue);
```

The value is stored in:

```tsx
myRef.current
```

Changing `.current` does not re-render the component.

Clean comparison:

```txt
useState:
remember this value and re-render when it changes

useRef:
remember this value but do not re-render when it changes
```

Common uses:

### DOM element access

```tsx
"use client";

import { useRef } from "react";

export default function SearchBox() {
  const inputRef = useRef<HTMLInputElement>(null);

  function focusInput() {
    inputRef.current?.focus();
  }

  return (
    <>
      <input ref={inputRef} />
      <button onClick={focusInput}>Focus input</button>
    </>
  );
}
```

### Prevent duplicate actions

```tsx
const hasSubmitted = useRef(false);

async function submitOrder() {
  if (hasSubmitted.current) return;

  hasSubmitted.current = true;
  await createOrder();
}
```

### Timer IDs

```tsx
const timeoutRef = useRef<NodeJS.Timeout | null>(null);

function startTimer() {
  timeoutRef.current = setTimeout(() => {
    console.log("done");
  }, 1000);
}

function cancelTimer() {
  if (timeoutRef.current) {
    clearTimeout(timeoutRef.current);
  }
}
```

For the storefront:

```txt
useState:
cart, filters, selected variation, modal open/closed

useRef:
input focus, scroll positions, timeout IDs, preventing duplicate checkout clicks
```

---

## 14. Client Provider with Server Children

A client provider can wrap server-rendered children without making all the children client components.

Correct pattern:

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

Provider:

```tsx
"use client";

export default function CartProvider({ children }: { children: React.ReactNode }) {
  return (
    <CartContext.Provider value={/* cart state */}>
      {children}
    </CartContext.Provider>
  );
}
```

This is okay because the server-rendered route content is passed as `children`.

Mental model:

```txt
Server renders the page/server children
  ↓
Client provider wraps the rendered output
  ↓
Browser receives HTML for server content
  ↓
Only client parts hydrate
```

Bundling:

```txt
Client provider JS is sent to the browser.
Server child component code is not sent to the browser.
Rendered HTML/result of server child is sent.
```

SEO:

```txt
Server children still appear in initial HTML.
So wrapping the app in a client cart provider does not automatically ruin SEO.
```

But server components cannot read client context:

```tsx
const cart = useCart(); // not allowed in a server component
```

Only client components inside the provider can use the cart context.

---

## 15. Server vs Client for SEO

Not all pages matter equally for SEO.

SEO-important pages:

```txt
home page
category pages
product detail pages
brand/collection pages
blog/content pages
```

Less SEO-important pages:

```txt
cart
checkout
login
account
admin dashboard
order confirmation
settings
bookmarks
```

Good rule:

```txt
Public discovery pages = server-first, SEO-friendly
Private/interactive utility pages = client-heavy is fine
```

For the ecommerce site:

```txt
Product grid page:
server component or server-fetched first batch

Product detail page:
server-rendered product info

Add to cart button:
client component

Cart drawer:
client component

Checkout session creation:
server action

Admin dashboard:
can be client-heavy because SEO does not matter
```

---

## 16. Can Client Components Contribute to SEO?

Yes, if they initially render with meaningful data.

A client component does not automatically mean:

```txt
empty HTML until JavaScript runs
```

In Next.js, client components can be pre-rendered into initial HTML when they receive data at render time.

SEO-friendly client component pattern:

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

If `initialProducts` is provided by the server page, the product names/links can be in the initial HTML.

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

This likely sends loading/empty HTML first, then fills the grid after JavaScript runs.

Better question than “is it client?”:

```txt
Are the product names, links, images, prices, and category text in the initial HTML?
```

If yes, it can contribute to SEO.

---

## 17. Product Grid Inside a Client Fetch More Component

Having a product grid inside a client `FetchMore` component is okay if the first batch comes from the server.

Bad pattern:

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

Initial HTML is empty or loading-only.

Good pattern:

```tsx
// app/category/[slug]/page.tsx - server component
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

Client component:

```tsx
"use client";

export default function FetchMoreProducts({ initialProducts, categorySlug }) {
  const [products, setProducts] = useState(initialProducts);
  const [page, setPage] = useState(1);

  async function loadMore() {
    const nextPage = page + 1;

    const res = await fetch(
      `/api/products?category=${categorySlug}&page=${nextPage}`
    );

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

Best ecommerce pattern:

```txt
Server renders page 1 products for SEO.
Client "Load more" fetches page 2+.
Optional real pagination URLs exist for crawlability.
```

Example URLs:

```txt
/category/sunglasses?page=1
/category/sunglasses?page=2
/category/sunglasses?page=3
```

---

## 18. URL Params vs State Filters

Filters should often live in the URL because it makes the filtered page state real and shareable.

Example:

```txt
/category/sunglasses?color=black&style=biker&inStock=true
```

Benefits:

```txt
shareable links
refresh keeps filters
browser back/forward works
server components can fetch filtered products
Google can discover useful filtered pages
users can land directly on filtered results
```

If filters only live in React state, the URL stays generic:

```txt
/category/sunglasses
```

Then the filtered data may only appear after client-side JavaScript runs.

Better server pattern:

```tsx
export default async function CategoryPage({ searchParams }) {
  const color = searchParams.color;
  const products = await getProducts({ color });

  return <ProductGrid products={products} />;
}
```

SEO strategy:

```txt
Important category/filter pages:
real URLs, server-rendered, maybe indexable

Random filter/sort combinations:
URL params for usability, but consider canonical/noindex
```

Useful search pages:

```txt
/motorcycle-sunglasses
/prescription-sunglasses
/category/sunglasses?lens=polarized
/category/sunglasses?color=black
/category/bifocals?strength=2.00
```

Less useful to index:

```txt
/category/sunglasses?sort=price_desc
/category/sunglasses?page=7&inStock=true&color=black&size=medium
```

Mental model:

```txt
State filters = temporary UI
URL filters = real page state, SEO, sharing, refresh, server fetching
```

---

## 19. Website Legal / Lawsuit-Risk Checklist

No website can be lawsuit-proof, but an ecommerce site should have basic policies, compliance, and honest product handling.

Core pages/policies:

```txt
Privacy Policy
Terms of Service / Terms of Use
Shipping Policy
Return / Refund Policy
Contact page / support email / business info
```

### Privacy Policy

Should explain:

```txt
what data is collected
why data is collected
who data is shared with
cookies/analytics
payment processors
user rights
contact info
```

### Terms of Service

Should cover:

```txt
acceptable use
account rules
pricing mistakes
order cancellation
limitation of liability
disputes/governing law
checkout agreement
```

Clickwrap-style agreement is stronger than only linking terms in the footer.

### Shipping Policy

Should cover:

```txt
where you ship
processing time
estimated delivery
shipping costs
tracking
lost packages
delays
```

### Return / Refund Policy

Should cover:

```txt
return window
item condition
who pays return shipping
final sale items
refund method
refund timing
```

### Product Accuracy

Product claims should be true and supportable, especially claims like:

```txt
UV protection
polarized
ANSI/safety rated
prescription compatible
scratch resistant
impact resistant
brand/origin claims
sale/compare-at pricing
```

### Sales Tax

For Texas ecommerce, sales tax likely matters for taxable goods. Use Stripe Tax, TaxJar, Avalara, or a similar setup if needed.

### Payment Security

Use Stripe Checkout or Stripe Elements instead of touching raw card data directly.

Good:

```txt
Stripe-hosted checkout
server action creates checkout session
Stripe handles card fields
webhook confirms payment
```

Avoid:

```txt
storing card numbers
processing raw card data yourself
collecting sensitive payment info directly
```

### Accessibility

Basic accessibility work:

```txt
semantic HTML
keyboard navigation
alt text for images
labels for inputs
visible focus states
reasonable contrast
screen-reader-friendly buttons and links
```

### Copyright and Trademarks

Only use content you own or are allowed to use:

```txt
product images
logos
brand names
descriptions
marketing copy
icons
fonts
videos
```

Do not scrape competitor images/copy.

### Email Marketing

Promotional email should include:

```txt
honest subject lines
clear sender identity
physical mailing address if required
unsubscribe link
no deceptive headers
```

### Reviews and Influencers

Avoid:

```txt
fake reviews
undisclosed paid/gifted endorsements
misleading testimonials
```

### Children’s Data

Do not knowingly collect personal data from children under 13 unless you are ready for COPPA compliance.

### Business Risk Reduction

Consider:

```txt
LLC or business entity
business bank account
business insurance
clear invoices/order records
registered agent/business address
```

---

## 20. Practical Architecture for the Storefront

Recommended split:

```txt
Public product/category pages:
server-rendered for SEO

First product batch:
fetched on the server

Load more:
client fetches next pages

Cart:
client context + localStorage persistence

Checkout:
server action creates Stripe Checkout session

Auth:
Supabase SSR cookie-based auth

Admin:
server actions with admin checks

Database:
server-side Supabase access

Legal/policy pages:
static server-rendered pages
```

Suggested structure:

```txt
app/
  layout.tsx
  page.tsx
  category/
    [slug]/
      page.tsx
      FetchMoreProducts.tsx
  product/
    [slug]/
      page.tsx
  login/
    page.tsx
    actions.ts
  cart/
    page.tsx
  checkout/
    actions.ts
  legal/
    privacy/
      page.tsx
    terms/
      page.tsx
    shipping/
      page.tsx
    returns/
      page.tsx

components/
  CartProvider.tsx
  ProductGrid.tsx
  ProductCard.tsx
  AddToCartButton.tsx
  CartDrawer.tsx
  Filters.tsx

lib/
  supabase/
    browser.ts
    server.ts
  stripe.ts
  products.ts
```

---

## 21. Key Final Mental Models

```txt
CORS:
browser rule controlling whether frontend JS can read cross-origin responses

Auth:
backend rule deciding who is allowed to access data

Server component:
runs on server, good for SEO/data/secrets, avoids CORS

Client component:
runs/hydrates in browser, good for interactivity/state/events/browser APIs

Server action:
hidden endpoint-like function; code/secrets stay server-side, but action still needs auth checks

Supabase SSR:
uses cookies so both browser and server can share auth session

Context:
shares state across client components

useEffect:
runs side effects after render

useRef:
stores values without causing re-renders

URL filters:
real page state, better for sharing, refresh, server fetching, and sometimes SEO

State filters:
temporary UI state, not ideal as the only source of product discovery state

SEO:
main question is whether important content is in the initial HTML

Legal:
clear policies, accurate claims, safe payments, accessibility, privacy, and honest marketing reduce risk
```

---

## 22. Current Direction

For the ecommerce storefront, the clean direction is:

```txt
Use server-rendered product/category pages.
Use a client FetchMore component with server-provided initial products.
Use URL params for important filters.
Use a cart provider for cart state.
Persist cart to localStorage.
Use Supabase SSR cookies for auth.
Use server actions for login, checkout, admin actions, and writes.
Protect every server action with auth/authorization checks.
Use Stripe-hosted checkout or Stripe Elements.
Add Privacy, Terms, Shipping, Return, and Contact pages.
Keep product claims accurate and supportable.
```

This setup keeps the public site SEO-friendly, the sensitive logic server-side, the cart interactive, and the legal/customer-facing basics covered.
