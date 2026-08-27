# frontend_architecture_and_claude_design_workflow.md (June 7, 2026)

*Last Updated: 2026-06-08*

---

# Goal

Build the BikerShades, Sunglass Monster, and ProSport storefronts on top of a clean backend and API contract rather than designing the frontend first and forcing the backend to match later.

The frontend should be generated and accelerated using Claude Design, while architecture, data modeling, and backend decisions remain under our control.

---

# Key Realization

The frontend should not drive the backend.

Bad workflow:

```text
Design page
    ↓
Generate frontend
    ↓
Figure out database later
    ↓
Rewrite frontend repeatedly
```

Good workflow:

```text
Data model
    ↓
Database
    ↓
API contract
    ↓
Populate data
    ↓
Generate frontend
    ↓
Integrate
```

The backend becomes stable before UI generation begins.

---

# Why Backend First

The migration project already established:

```text
Products
Variants
Categories
Images
Brands
```

as the core entities.

The UI should consume those entities rather than inventing new structures.

Without a defined API:

```text
Claude invents data shape A
Backend implements data shape B
Integration becomes painful
```

With a defined API:

```text
Backend defines shape
Claude consumes shape
Integration becomes straightforward
```

---

# Proposed Architecture

```text
Amazon
eBay
      ↓
    Veeqo
      ↓
   Supabase
      ↓
Next.js Storefronts
```

Inventory management remains inside:

```text
Amazon
eBay
Veeqo
```

Storefronts become consumers of inventory data rather than the source of truth.

---

# Frontend Stack

## Framework

```text
Next.js App Router
```

Reason:

* SEO
* Server Components
* URL-based navigation
* Vercel support
* Modern React architecture

---

## Styling

```text
Tailwind CSS
```

Tailwind becomes the single styling system.

Avoid:

```text
Raw CSS files
CSS Modules
Styled Components
Emotion
```

as the primary styling strategy.

---

## Components

```text
shadcn/ui
```

Used for:

* Buttons
* Cards
* Dialogs
* Tabs
* Navigation Menu
* Sheet
* Accordion
* Select
* Checkbox
* Carousel

---

# Important shadcn/ui Discovery

shadcn components are copied into the project.

Example:

```bash
npx shadcn add button
```

creates:

```text
components/ui/button.tsx
```

inside the repository.

This means:

```text
We own the code.
```

Unlike traditional component libraries:

```text
node_modules
```

is not the source of truth.

The component implementation lives in the project itself.

Benefits:

* modify behavior
* modify styles
* modify animations
* modify accessibility
* create custom variants

without fighting a third-party dependency.

---

# Claude Design Workflow

Claude Design should NOT build the entire application.

Avoid prompts like:

```text
Build an ecommerce store.
```

Instead:

```text
Build a reusable product page component.
Build a reusable homepage.
Build a reusable navbar.
Build a reusable category page.
```

Focus on isolated UI pieces.

---

# What Claude Design Should Build

Examples:

```text
Homepage
Product Page
Category Page
Product Card
Mega Menu
Footer
Mobile Navigation
Search Overlay
Filter Sidebar
Cart Drawer
```

---

# What Claude Design Should NOT Build

Avoid asking Claude Design to design:

```text
Database schema
Inventory synchronization
Veeqo integration
Category hierarchy logic
Search architecture
API design
Authentication
```

Those are architecture problems, not design problems.

---

# API-Driven Frontend Generation

Before generating UI:

Define endpoints.

Examples:

```text
GET /api/categories

GET /api/categories/[slug]

GET /api/products/[slug]

GET /api/products?category=

GET /api/search?q=
```

Define exact response shapes.

Example:

```json
{
  "id": "123",
  "slug": "eliminator-night-vision-bifocal",
  "name": "Eliminator Night Vision Bifocal",
  "price": 29.99,
  "images": [],
  "variants": []
}
```

Once the contract exists:

Claude Design can build directly against it.

---

# Recommended Claude Design Prompt Pattern

```text
Stack:
- Next.js App Router
- Tailwind CSS
- shadcn/ui

Use the provided API contract.

Use Server Components.

Do not create your own API layer.
Do not invent data structures.
Do not build a full project.

Build only reusable UI components.
```

---

# Tailwind Requirement

Previous attempts generated:

```text
Raw CSS files
Custom styling systems
Hardcoded styles
```

which caused integration problems.

Future prompts should require:

```text
Tailwind only.
```

Example:

Good:

```tsx
<Card className="rounded-xl p-4">
```

Bad:

```css
.product-card {
  border-radius: 12px;
}
```

---

# Data Flow Philosophy

The storefront should primarily use:

```text
Server Components
```

instead of client-side fetching.

Example:

```tsx
const product = await getProduct(slug);
```

rather than:

```tsx
useEffect(() => {
  fetch(...)
})
```

This provides:

* SEO
* simpler architecture
* smaller client bundles
* cleaner data flow

---

# URL-Based Navigation

State should primarily exist in URLs.

Good:

```text
/products/eliminator-night-vision-bifocal

/categories/sport-bifocals

/search?q=aviator

/categories/polarized?color=black
```

Bad:

```text
Everything hidden in React state
```

Benefits:

* shareable links
* SEO
* browser history support
* deep linking
* crawlability

---

# Suspense Learnings

Without Suspense:

```text
await data
    ↓
render page
```

Everything waits.

---

With Suspense:

```text
render page shell
    ↓
show fallback
    ↓
replace fallback when data arrives
```

Suspense allows:

```text
Navbar
Footer
Layout
```

to appear while slower sections continue loading.

Examples:

```text
Product Grid
Recommendations
Reviews
Search Results
```

can be wrapped in Suspense boundaries.

---

# Important Suspense Rule

Anything awaited before the return statement blocks the entire page.

Example:

```tsx
const products = await getProducts();
```

at the top level prevents rendering until complete.

Suspense only helps when the fetch lives inside the boundary.

Example:

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

---

# Final Workflow

```text
1. Finish migration

2. Finalize data model

3. Create Supabase schema

4. Populate catalog

5. Define API contract

6. Deploy endpoints

7. Use Claude Design to build:
      - Homepage
      - Product Page
      - Category Page
      - Navbar
      - Product Cards
      - Filters

8. Integrate generated UI

9. Connect real API data

10. Launch storefronts
```

---

# Core Principle

Claude Design should be treated as a frontend designer.

Claude Code should be treated as an implementation assistant.

Architecture decisions remain under our control.

```text
You
    ↓
Data Model

You
    ↓
API Contract

Claude Design
    ↓
UI

Claude Code
    ↓
Integration
```

This separation minimizes rewrites and keeps the project maintainable as the catalog grows.

