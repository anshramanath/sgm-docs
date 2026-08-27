# Storefront Architecture Notes (June 16, 2026)

## React State, Providers, Local Storage, Server Components, and Stripe Checkout

---

# Cart Storage Strategy

## Decision

Store the cart in localStorage.

Reasoning:

* Works for guests.
* Persists between page refreshes.
* Persists after closing browser.
* No authentication required.
* No database writes.
* Fast.

Example:

```ts
type CartItem = {
  productId: string;
  variationId: string;
  quantity: number;
};
```

Store only:

* productId
* variationId
* quantity

Do NOT store:

* product name
* images
* stock
* prices
* tax

Those should always come from the backend.

---

# Cart Retrieval Flow

```text
localStorage
    ↓
load identifiers
    ↓
fetch latest product data
    ↓
render cart
```

Benefits:

* Reflects current inventory.
* Reflects current pricing.
* Handles deleted products.
* Handles updated images.

---

# Future Customer Accounts

Even after authentication is added:

```text
localStorage
+
database cart
```

Recommended architecture:

```text
Guest cart
    ↓
Login
    ↓
Merge local + database cart
    ↓
Save merged cart
```

Local storage should remain.

Benefits:

* Faster loads.
* Offline resilience.
* Less database traffic.
* Better user experience.

---

# Why Context Providers Exist

Without context:

```text
App
 ↓
Layout
 ↓
Page
 ↓
ProductCard
 ↓
AddToCartButton
```

Cart props must be passed through every level.

This is prop drilling.

---

With context:

```tsx
<CartProvider>
  {children}
</CartProvider>
```

Any child can access:

```ts
const {
  cart,
  addToCart
} = useCart();
```

without receiving props.

---

# Recommended Providers

## CartProvider

Owns:

```ts
cart
addToCart()
removeFromCart()
updateQuantity()
clearCart()
cartCount
```

Persists cart to localStorage.

---

## WishlistProvider

Owns:

```ts
bookmarks
addBookmark()
removeBookmark()
isBookmarked()
```

---

## CategoryProvider

Owns:

```ts
categoryTree
flatCategoryMap
loading
```

Useful for:

* navbar
* breadcrumbs
* category pages
* product pages

---

# Provider Placement

Recommended:

```tsx
app/layout.tsx
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

Site-wide state belongs in layouts.

---

# Why Providers Use useState

Bad:

```ts
let cart = [];
```

React does not know when this changes.

Good:

```tsx
const [cart, setCart] = useState([]);
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

# Understanding useEffect

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
render
    ↓
dependency comparison
    ↓
effect runs if changed
```

---

# Why State Variables Work Better

Example:

```tsx
const [cart, setCart] = useState([]);
```

Flow:

```text
setCart()
    ↓
rerender
    ↓
dependency check
    ↓
effect runs
```

Module variables:

```ts
let cart = [];
```

Flow:

```text
cart changes
    ↓
nothing
```

React is never informed.

---

# Dependency Arrays

## Empty Array

```tsx
useEffect(() => {
}, []);
```

Runs:

```text
Mount only
```

---

## With Dependencies

```tsx
useEffect(() => {
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

Lifecycle:

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
Cleanup runs before
the effect reruns.

Cleanup also runs
on unmount.
```

---

# Empty Dependency Arrays and Cleanup

Without return:

```tsx
useEffect(() => {
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

---

With return:

```tsx
useEffect(() => {
  return () => {
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

Key takeaway:

```text
The effect body never runs on unmount.

Only the cleanup function runs on unmount.
```

---

# Next.js Server Components

Server components are:

```text
Fetch
Compute
Render
Deliver
```

Example:

```tsx
const product =
  await getProduct();
```

This is good.

Why?

Because the value is resolved on the server.

The browser receives the finished result.

---

# Client Components

Client components use:

```tsx
"use client";
```

Can use:

```tsx
useState()
useEffect()
useContext()
```

Responsible for:

* cart
* wishlist
* localStorage
* modals
* dropdowns
* interactive UI

---

# Important Discovery

Client Providers Do NOT Make Children Client Components

Example:

```tsx
<Providers>
  <ProductPage />
</Providers>
```

ProductPage can remain:

```tsx
Server Component
```

The provider receives already-rendered server output.

---

# What Server Components Cannot Do

Invalid:

```tsx
const { cart } = useCart();
```

inside:

```tsx
export default async function ProductPage() {}
```

Server components cannot use:

```tsx
useState()
useEffect()
useContext()
useCart()
```

---

# Correct Pattern

Server:

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

Client:

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

# Mental Model

Server Components:

```text
Create once
Deliver
```

Client Components:

```text
Stay alive
React to user interaction
Manage state
```

---

# Stripe Checkout Decision

## Chosen Approach

Use Stripe Checkout Sessions.

Not:

```text
Payment Links
```

and not:

```text
Custom Stripe Elements Checkout
```

---

# Why Stripe Checkout

Flow:

```text
Cart
 ↓
Checkout Button
 ↓
Create Session
 ↓
Redirect to Stripe
 ↓
Payment
 ↓
Webhook
 ↓
Order Created
```

Benefits:

* Apple Pay
* Google Pay
* Tax handling
* Address collection
* PCI compliance
* 3D Secure
* Fraud protection

---

# Why Custom Checkout Is Hard

The UI is easy.

The infrastructure is hard.

Custom checkout requires:

```text
Address validation
Shipping selection
Tax calculation
Payment forms
Card validation
3D Secure
Payment errors
Order creation
Inventory updates
Email confirmations
Webhook handling
Retry handling
Duplicate protection
```

The screenshots we designed are visually close to a custom checkout already.

However the complexity exists behind the button.

---

# Checkout UX Direction

Instead of:

```text
Embedded Card Form
```

Use:

```text
Checkout Review Page
```

Customer sees:

* products
* quantities
* subtotal
* shipping
* estimated tax
* total estimate

Then clicks:

```text
Continue to Secure Checkout
```

and is redirected to Stripe.

---

# Tax Handling

Important discovery:

Final tax usually cannot be calculated until Stripe knows:

```text
Country
State
City
ZIP Code
```

Therefore:

Cart page should show:

```text
Taxes calculated during checkout.
```

or

```text
Estimated tax calculated at secure checkout.
```

Flow:

```text
Cart
 ↓
Stripe Checkout
 ↓
Customer enters address
 ↓
Stripe calculates tax
 ↓
Final total shown
 ↓
Payment
```

Recommended V1 approach:

Let Stripe handle:

* tax calculation
* tax collection
* tax reporting

Do not build custom tax logic initially.

---

# Final Storefront Architecture

```text
Server Components
-----------------
Home Page
Category Page
Product Page
Search Page

Client Components
-----------------
Cart Provider
Wishlist Provider
Search Bar
Add To Cart Button
Quantity Selector
Mobile Menu

Persistence
-----------
localStorage cart
localStorage wishlist

Payments
--------
Stripe Checkout Session

Backend
-------
Validate cart
Create checkout session
Receive webhook
Create order

Taxes
-----
Calculated by Stripe
during checkout
```

