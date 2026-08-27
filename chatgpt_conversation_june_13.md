# React State, Context Providers, Local Storage, and Next.js Server Components (June 13, 2026)

## Cart Storage Strategy

### Initial Decision

Store the shopping cart in localStorage.

Benefits:

* Works without authentication.
* Persists across page refreshes.
* Persists across browser restarts.
* No database writes required.
* Fast access.
* Simple implementation.

Example cart:

```ts
type CartItem = {
  productId: string;
  variationId: string;
  quantity: number;
};
```

Only store identifiers and quantities.

Do not store:

* price
* stock
* images
* product name

These should always come from the backend.

### Cart Load Flow

```text
localStorage
    ↓
load cart ids
    ↓
fetch latest product data
    ↓
render cart
```

This protects against:

* price changes
* inventory changes
* deleted products
* image updates

### Future Account Support

Even after customer accounts exist:

```text
localStorage = browser cache

database = account-wide cart
```

Recommended flow:

```text
Guest cart
    ↓
User logs in
    ↓
Merge local cart + database cart
    ↓
Save merged result to database
    ↓
Continue syncing
```

Keep localStorage even with accounts because:

* Faster initial load.
* Works offline.
* Prevents cart loss during auth issues.
* Supports guests.

---

# localStorage Basics

Save:

```ts
localStorage.setItem(
  "cart",
  JSON.stringify(cart)
);
```

Load:

```ts
const cart =
  JSON.parse(
    localStorage.getItem("cart") ?? "[]"
  );
```

Remove:

```ts
localStorage.removeItem("cart");
```

Clear everything:

```ts
localStorage.clear();
```

---

# Why Context Providers Exist

Problem:

```text
Navbar needs cart count
Product page adds items
Cart page edits quantities
Checkout reads cart
```

Without Context:

```text
App
 ↓
Layout
 ↓
Page
 ↓
Button
```

Cart props must be passed through every layer.

This is called prop drilling.

### Context Solution

```tsx
<CartProvider>
  {children}
</CartProvider>
```

Now any descendant can access:

```ts
const {
  cart,
  addToCart
} = useCart();
```

No prop drilling required.

---

# Cart Provider Responsibilities

Provider owns:

```ts
cart
addToCart()
removeFromCart()
updateQuantity()
clearCart()
cartCount
```

Provider should also synchronize localStorage.

Example:

```tsx
useEffect(() => {
  localStorage.setItem(
    "cart",
    JSON.stringify(cart)
  );
}, [cart]);
```

---

# Why Providers Use State

Bad:

```ts
let cart = [];
```

React does not know when this changes.

Good:

```tsx
const [cart, setCart] =
  useState([]);
```

React knows:

```text
setCart()
    ↓
rerender
    ↓
UI updates
```

---

# useEffect Mental Model

Most important rule:

```text
useEffect does not watch variables.

useEffect checks dependencies
after renders.
```

Incorrect mental model:

```text
variable changes
    ↓
effect runs
```

Correct mental model:

```text
render happens
    ↓
dependencies compared
    ↓
effect runs if changed
```

---

# Dependency Arrays

## Empty Array

```tsx
useEffect(() => {
  ...
}, []);
```

Runs:

```text
Mount only
```

Does not run again.

---

## Dependency Array

```tsx
useEffect(() => {
  ...
}, [cart]);
```

Runs:

```text
Mount
Cart changes
Cart changes
Cart changes
...
```

Every render where cart differs from previous render.

---

# Cleanup Functions

Example:

```tsx
useEffect(() => {
  setup();

  return () => {
    cleanup();
  };
}, [cart]);
```

Flow:

```text
Mount
 ↓
setup

cart changes
 ↓
cleanup
 ↓
setup

cart changes
 ↓
cleanup
 ↓
setup

Unmount
 ↓
cleanup
```

Rule:

```text
Cleanup runs before the effect reruns.

Cleanup also runs on unmount.
```

---

# Empty Dependency Arrays and Cleanup

Without cleanup:

```tsx
useEffect(() => {
  ...
}, []);
```

Flow:

```text
Mount
 ↓
effect

Unmount
 ↓
nothing
```

With cleanup:

```tsx
useEffect(() => {
  return () => {
    ...
  };
}, []);
```

Flow:

```text
Mount
 ↓
effect

Unmount
 ↓
cleanup
```

Therefore:

```text
Effect body runs on mount.

Cleanup runs on unmount.
```

---

# Provider Placement

Best practice:

```tsx
app/layout.tsx
```

Wrap application:

```tsx
<Providers>
  {children}
</Providers>
```

Example:

```tsx
<CategoryProvider>
  <CartProvider>
    <WishlistProvider>
      {children}
    </WishlistProvider>
  </CartProvider>
</CategoryProvider>
```

Use a provider when many distant parts of the app need the same state.

Examples:

✅ Cart

✅ Wishlist

✅ Categories

Not good provider candidates:

❌ Single page form state

❌ Local modal state

❌ Single component state

---

# Next.js Server vs Client Components

## Server Component

Runs on server.

Example:

```tsx
const product =
  await getProduct();
```

Good because:

```text
Fetch data
Render page
Send result
Done
```

Server components are mostly:

```text
Create once
Deliver
```

---

## Client Component

Uses:

```tsx
"use client";
```

Can use:

```tsx
useState()
useEffect()
useContext()
```

Handles:

* cart
* wishlist
* modals
* dropdowns
* search state
* localStorage

---

# Important Discovery

Client Providers Do NOT Make Children Client Components

Example:

```tsx
<Providers>
  <ProductPage />
</Providers>
```

ProductPage can still be a Server Component.

Provider receives already-rendered server content.

---

# Server Components Cannot Use Context

Bad:

```tsx
const { cart } = useCart();
```

inside:

```tsx
export default async function ProductPage() {}
```

Server Components cannot use:

```tsx
useState()
useEffect()
useContext()
useCart()
```

---

# Correct Pattern

Server Page:

```tsx
export default async function ProductPage() {
  const product =
    await getProduct();

  return (
    <>
      <ProductInfo />
      <AddToCartButton />
    </>
  );
}
```

Client Component:

```tsx
"use client";

const { addToCart } =
  useCart();
```

Architecture:

```text
ProductPage (server)
    ↓
AddToCartButton (client)
    ↓
CartProvider (client)
```

---

# Final Mental Model

Server Components:

```text
Fetch
Compute
Render
Deliver
```

Client Components:

```text
Interactive
Reactive
Stateful
Persistent
```

Server Components handle product data.

Client Components handle cart state.

Server Components are "create once and deliver."

Client Components stay alive in the browser and react to user actions.

