# frontend_architecture_react_nextjs_learnings.md (June 11, 2026)

*Last Updated: 2026-06-11*

---

# Goal

Design a scalable architecture for the new BikerShades, Sunglass Monster, and ProSport storefronts while avoiding the common pitfalls of frontend-heavy ecommerce applications.

The goal is to:

```text
Build backend first
Define API contracts
Generate UI against stable contracts
Avoid frontend/backend mismatches
Maintain SEO and performance
```

---

# Storefront Stack Decision

Chosen stack:

```text
Next.js App Router
Tailwind CSS
shadcn/ui
Supabase
Vercel
```

Reasoning:

* SEO-friendly
* Server rendering
* URL-based navigation
* Modern React architecture
* Fast development
* Easy deployment
* Works well with AI-generated UI

---

# Tailwind vs shadcn/ui

## Tailwind

Tailwind is the styling system.

Example:

```tsx
<div className="flex gap-4 rounded-lg p-6">
```

Tailwind handles:

* spacing
* colors
* typography
* responsiveness
* layout

---

## shadcn/ui

shadcn/ui provides reusable React components.

Examples:

```text
Button
Card
Dialog
Tabs
Accordion
NavigationMenu
Sheet
Select
Checkbox
Carousel
```

shadcn components are built using Tailwind.

---

## Major shadcn Discovery

Unlike traditional UI libraries:

```text
node_modules
```

is NOT the source of truth.

Running:

```bash
npx shadcn add button
```

creates:

```text
components/ui/button.tsx
```

inside the project.

Meaning:

```text
We own the code.
```

Benefits:

* modify styling
* modify behavior
* add variants
* add animations
* customize freely

without fighting a third-party library.

---

# Claude Design Workflow

Claude Design should NOT build an entire application.

Bad prompt:

```text
Build an ecommerce store.
```

Good prompts:

```text
Build a homepage component.
Build a product page component.
Build a mega menu component.
Build a category page component.
```

Claude Design should focus on:

```text
UI
Layout
User experience
Component composition
```

not:

```text
Database architecture
Authentication
Inventory synchronization
API design
```

---

# Backend First Architecture

Major realization:

The frontend should not dictate backend structure.

Bad workflow:

```text
Generate frontend
↓
Figure out backend later
↓
Rewrite everything
```

Preferred workflow:

```text
Data model
↓
Database
↓
API contract
↓
Populate data
↓
Generate UI
↓
Integration
```

---

# API Contract First

Before generating UI:

Define endpoints.

Examples:

```text
GET /api/brands/[brandSlug]/home

GET /api/brands/[brandSlug]/categories

GET /api/brands/[brandSlug]/products

GET /api/brands/[brandSlug]/products/[slug]

GET /api/search
```

Define exact response shapes.

Example:

```json
{
  "id": "...",
  "slug": "...",
  "name": "...",
  "price": 29.99,
  "images": [],
  "variants": []
}
```

Then Claude Design consumes those contracts.

---

# URL-Based Navigation

Important realization:

```text
URL = state
```

Good:

```text
/products/eliminator-night-vision-bifocal

/categories/sport-bifocals

/search?q=aviator

/categories/polarized?color=black
```

Bad:

```text
Everything stored only in React state.
```

Benefits:

* SEO
* shareable links
* browser history support
* crawlability
* deep linking

---

# Server Components

Most pages should be server rendered.

Example:

```tsx
const product = await getProduct(slug);
```

instead of:

```tsx
useEffect(() => {
  fetch(...)
})
```

Benefits:

```text
SEO
simpler architecture
smaller bundles
better performance
```

---

# Client Components

Client components are still useful for:

```text
Mega menu interactions
Mobile navigation
Image galleries
Variant selection
Search inputs
Filter UI
Cart interactions
```

The key insight:

```text
Server Components fetch data.

Client Components manage interactions.
```

---

# Suspense Learnings

## Without Suspense

```tsx
const products = await getProducts();

return (...)
```

Flow:

```text
Wait for products
↓
Render everything
```

Entire page is blocked.

---

## With Suspense

```tsx
<Suspense fallback={<Skeleton />}>
  <ProductsSection />
</Suspense>
```

Flow:

```text
Render page shell
↓
Show fallback
↓
Products arrive
↓
Replace fallback
```

Benefits:

```text
Navbar appears immediately
Layout appears immediately
Products stream in later
```

---

# Important Suspense Rule

Suspense only helps if the fetch happens inside the boundary.

Bad:

```tsx
const products = await getProducts();

return (
  <Suspense>
    <Products />
  </Suspense>
)
```

Still blocked.

---

Good:

```tsx
<Suspense fallback={<Skeleton />}>
  <ProductsSection />
</Suspense>
```

where:

```tsx
async function ProductsSection() {
  const products = await getProducts();
}
```

Now only the boundary waits.

---

# React useEffect Learnings

## Effect Lifecycle

Structure:

```tsx
useEffect(() => {
  // setup

  return () => {
    // cleanup
  };
}, deps);
```

Mental model:

```text
Setup
↓
Dependency changes
↓
Cleanup
↓
Setup again
```

---

# Cleanup Functions

React does not magically clean things up.

Instead:

```text
React calls the cleanup function we provide.
```

Examples:

```text
clearTimeout
removeEventListener
socket.close
clearInterval
unsubscribe
```

---

# Debouncing

Definition:

```text
Wait until the user stops doing something
before running expensive logic.
```

Example:

Search input.

Without debounce:

```text
a
av
avi
avia
aviat
aviato
aviator
```

Results in:

```text
7 API calls
```

---

With debounce:

```text
User types
↓
Timer resets
↓
User stops typing
↓
Single API call
```

---

# How Debouncing Works

Example:

```tsx
useEffect(() => {
  const timer = setTimeout(() => {
    search();
  }, 300);

  return () => clearTimeout(timer);
}, [searchTerm]);
```

Flow:

```text
Type
↓
Create timer

Type again
↓
Cleanup runs
↓
Clear old timer
↓
Create new timer

Stop typing
↓
Timer finishes
↓
Run search
```

---

# Key Discovery

React is not cancelling timers.

The browser owns timers.

React simply:

```text
Runs cleanup before rerunning the effect.
```

Our cleanup calls:

```text
clearTimeout(timer)
```

which removes the timer from the browser.

---

# Dependency Arrays

## Empty Array

```tsx
useEffect(..., [])
```

Meaning:

```text
Run on mount
Cleanup on unmount
```

Example:

```text
Open websocket
Register listener
Start interval
```

Cleanup:

```text
Close websocket
Remove listener
Clear interval
```

---

## Dependency Array

```tsx
useEffect(..., [userId])
```

Meaning:

```text
Run on mount

Run again when userId changes

Cleanup before rerunning
```

---

## No Dependency Array

```tsx
useEffect(...)
```

Meaning:

```text
Run after every render
Cleanup before every rerun
```

Usually undesirable for expensive operations.

---

# WebSocket Realization

This:

```tsx
useEffect(() => {
  const socket = connect();

  return () => socket.close();
}, []);
```

does NOT reconnect every render.

Flow:

```text
Mount
↓
Open socket

Re-renders
↓
Nothing

Re-renders
↓
Nothing

Unmount
↓
Close socket
```

Because:

```text
[]
```

means:

```text
Mount + Unmount lifecycle
```

not:

```text
Every render lifecycle
```

---

# Product Filtering Architecture

Major realization:

Filtering should not happen entirely in the frontend.

Bad:

```text
Send entire catalog
↓
Filter client-side
```

Problems:

* large payloads
* poor performance
* mobile issues

---

Good:

```text
Frontend manages filter UI
↓
Backend filters data
↓
Frontend renders results
```

Example:

```text
/products?category=bifocals&page=2
```

---

# Pagination

Supabase provides pagination.

Example:

```ts
.range(0, 23)
```

returns:

```text
Rows 0 through 23
```

Range is:

```text
0-indexed
inclusive
```

---

Example:

```ts
.range(24, 47)
```

returns:

```text
24 rows
```

---

# Count

Example:

```ts
.select("*", { count: "exact" })
```

Returns:

```text
data = current page
count = total matching rows
```

Example:

```text
24 products returned
517 total matching products
```

Useful for:

```text
Pagination
Page counts
Result counts
```

---

# Product Filtering Metadata

Important discovery:

A category page needs:

```text
Current page products
+
Global filter metadata
```

Example:

```json
{
  "products": [...24 products],
  "pagination": {...},
  "facets": {
    "colors": [...],
    "lensTypes": [...],
    "price": {
      "min": 12.99,
      "max": 79.99
    }
  }
}
```

The facet metadata should be computed from ALL matching products, not only the current page.

This prevents:

```text
Wrong min/max price
Wrong filter counts
Incomplete filter options
```

---

# Final Architecture

```text
Veeqo
   ↓
Sync Jobs
   ↓
Supabase

Supabase
   ↓
API Endpoints

API Endpoints
   ↓
Next.js

Next.js
   ↓
Tailwind
   ↓
shadcn/ui
```

Frontend:

```text
UI
Interactions
Layout
```

Backend:

```text
Filtering
Searching
Pagination
Data retrieval
```

This provides a scalable architecture for all three storefronts.

