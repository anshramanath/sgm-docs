# Biker Shades Admin — Stack & Design Notes (May 31, 2026)

## Stack

| Layer | Choice |
|---|---|
| Framework | Next.js 16 (App Router) |
| Language | TypeScript |
| Graph canvas | @xyflow/react (React Flow) |
| Layout engine | dagre |
| Database | Supabase (Postgres) |
| Auth | Supabase Auth |
| Styling | Inline styles + CSS custom properties |
| Design system | Profiler (Source Serif 4 + Courier Prime) |

---

## Routes

| Route | Type |
|---|---|
| `/login` | Client page |
| `/admin` | Client page |
| `/admin/products/[id]` | Server page |
| `GET /api/admin/navigation-tree` | API route |
| `GET /api/admin/categories/[id]/products` | API route |
| `GET /api/admin/products/[id]` | API route |

---

## Design Decisions

**Profiler design system, not Tailwind.**
All styling uses inline styles with CSS custom properties (`var(--font-mono)`, `var(--border-color)`, etc.) defined in `globals.css`. Tailwind is installed but unused. This keeps the Profiler token system as the single source of truth.

**Two Supabase clients.**
The browser client (anon key + cookies) handles auth only. A separate service role client bypasses RLS for all data reads on the server. This avoids writing RLS policies for an admin-only tool while keeping auth secure.

**No function references in React Flow node data.**
React Flow's internal state management can detach functions stored in node `data`. Clicks on `ProductCountNode` are handled via `onNodeClick` on the `ReactFlow` component itself, which always holds a fresh reference to the callback.

**dagre for tree layout.**
Each top-level category tree is laid out independently with dagre (`rankdir: TB`), then the resulting trees are placed side by side with a fixed horizontal gap. This prevents trees from overlapping without any manual positioning.

**Inline product panel, not a modal.**
When a product count node is clicked, the product panel slides up from the bottom of the screen as an inline flex child — the React Flow canvas shrinks to make room. No overlay, no backdrop, no z-index stacking. The animation is a single CSS `height` transition with `cubic-bezier(0.4,0,0.2,1)`.

**Client-side pagination.**
Products per category are fetched in one request and paginated in the UI (12 per page). Categories rarely exceed a few hundred products, so this avoids the complexity of server-side cursor pagination on a joined query.

**Proxy (middleware) is auth-only.**
The proxy checks for a valid Supabase session and redirects to `/login` if absent. The `admins` table check was removed from the proxy because it requires the service role key, which cannot safely run in the Edge Runtime.

---

## Key Files

```
proxy.ts                          Auth guard (session check only)
app/admin/page.tsx                React Flow canvas + product panel
app/admin/products/[id]/page.tsx  Product detail (server component)
components/flow/CatalogFlow.tsx   ReactFlow wrapper, handles onNodeClick
components/flow/CategoryNode.tsx  Black root / white child nodes
components/flow/ProductCountNode  Grey clickable count nodes
components/products/ProductDrawer Sliding panel with pagination
lib/navigation/buildTree.ts       Assembles CategoryTreeNode hierarchy
lib/navigation/layoutTrees.ts     dagre layout per tree
lib/supabase/server.ts            createClient (auth) + createServiceClient (data)
```
