# Admin Panel — Microfrontend E-commerce

A minimal admin panel for an e-commerce platform, built as a **Microfrontend** architecture using **Module Federation 2.0**.

## Overview

The app is split into a host shell and several independent remotes, wired together at runtime via Module Federation. Each app under `apps/` is fully self-contained (its own `package.json` and `node_modules`) and is meant to eventually live in its own repository.

| Role | App | Port | Description |
|------|-----|------|-------------|
| Host | `apps/shell` | 3000 | Main app — sidebar, routing, renders remote modules |
| Remote | `apps/products` | 3001 | Product list — pagination + search + category filter (emits `productSelected`) |
| Remote | `apps/categories` | 3002 | Category panel — category tree + details |
| Remote | `apps/category-picker` | 3003 | Reusable category picker component (`./CategoryPicker`), consumed by other MFEs |
| Remote | `apps/product-page` | 3004 | Single product card — fetches a product by SKU |
| Shared | `apps/event-bus` | — | `mitt` singleton (`@admin/event-bus` package) for cross-MFE communication |

## Tech Stack

- **React 19** + TypeScript (strict mode)
- **Webpack 5 + `@module-federation/enhanced`** — Module Federation 2.0 (`react` / `react-dom` shared as `singleton`)
- **Tailwind CSS** — each MFE has its own config; shell loads global styles
- **mitt** — event bus (in `event-bus`)
- **react-router-dom v7** — URL routing; paths owned by the shell, query params also driven by `products`
- No Vite — full Webpack 5 with the MF plugin

## Project Structure

```
admin/
├── apps/                  # CODE — each subfolder is a self-contained component
│   ├── shell/             # Host — main shell with sidebar menu
│   ├── products/          # Remote — product LIST (consumes category-picker)
│   ├── product-page/      # Remote — single product CARD (./App, fetch by SKU)
│   ├── categories/        # Remote — categories microfrontend
│   ├── category-picker/   # Remote — reusable category picker (./CategoryPicker)
│   └── event-bus/         # Shared contract — mitt event bus (shared singleton)
├── docs/                  # Documentation (repo-split.md, ...)
├── AGENTS.md
└── README.md
```

There is no top-level npm workspace — each component in `apps/` installs and builds on its own
(`cd apps/<x> && npm install`). `event-bus` is consumed via `file:../event-bus`.

## Getting Started

Each component is standalone — install and run it separately (as if it were its own repo).

```bash
# Backend (in a separate terminal)
cd ../backend && ./run.sh start

# Each component in its own terminal (example: shell)
cd apps/shell && npm install && npm run dev
# likewise: apps/products, apps/categories, apps/category-picker, apps/product-page
```

To run everything at once you can use your own script / `concurrently`, or a thin meta-repo
(see `docs/repo-split.md`). Start order does not matter — remotes connect to the shell at runtime.

## Module Federation 2.0

The plugin comes from `@module-federation/enhanced` (MF 2.0), not the built-in `webpack.container`:

```ts
import { ModuleFederationPlugin } from "@module-federation/enhanced/webpack";
```

`dts: false` — remote types are declared manually in `shell/src/types.d.ts` (no automatic generation).

**Host (`shell`)** declares the remotes and shared singletons:

```ts
new ModuleFederationPlugin({
  name: "shell",
  remotes: {
    products: "products@http://localhost:3001/remoteEntry.js",
    categories: "categories@http://localhost:3002/remoteEntry.js",
    productPage: "productPage@http://localhost:3004/remoteEntry.js",
  },
  shared: {
    react: { singleton: true, requiredVersion: deps.react },
    "react-dom": { singleton: true, requiredVersion: deps["react-dom"] },
    // Without a singleton each MFE gets its own mitt instance -> events don't propagate
    "@admin/event-bus": { singleton: true, requiredVersion: false },
    // One router for the whole app (single history/context)
    "react-router-dom": { singleton: true, requiredVersion: deps["react-router-dom"] },
  },
  dts: false,
})
```

**Remotes (`products`, `categories`)** expose `./App` and share the same singletons.

The shell loads remotes via `React.lazy` + dynamic `import` (no `window` globals). `RemoteModule`
maps the active tab to a lazy component wrapped in an `ErrorBoundary` (fallback when a remote is
offline); `Suspense` lives in `App.tsx`.

### Async boundary (bootstrap)

`shared` singletons require asynchronous share-scope init, so each app's entry is split:

- `src/main.tsx` (shell) / `src/index.ts` (remote) — only `import("./bootstrap")`
- `src/bootstrap.tsx` — the actual `ReactDOM.createRoot(...).render(<App />)`

Remotes mount to `#root` in standalone mode and expose only `./App` as a remote. CSS is imported in
`App.tsx` (not in bootstrap) so styles travel with the exposed module and work in the host too.

### Category Picker (reusable remote)

`category-picker` (port `:3003`) exposes **`./CategoryPicker`** — a single component for multiple
use-cases. It is consumed like any remote, but it is a **component with props** (not a full page), so
communication happens through an `onSelect` callback rather than the event bus:

```tsx
const CategoryPicker = React.lazy(() => import("categoryPicker/CategoryPicker"));
// wrapped in ErrorBoundary + Suspense (fallback when the picker is offline)
<CategoryPicker selectionMode="any" onSelect={(sel) => ...} onCancel={...} />
```

Props (types declared manually in `products/src/types.d.ts`, `dts: false`):

| Prop | Type | Description |
|------|------|-------------|
| `onSelect` | `(sel: { id; name; path }) => void` | Returns the chosen category + breadcrumb path |
| `onCancel?` | `() => void` | Renders a "Cancel" button |
| `selectionMode?` | `"leaf" \| "any"` | `leaf` — only leaves selectable (e.g. changing a product's category); `any` — also the current branch (filter; backend filters subcategories anyway). Default `leaf` |
| `confirmLabel?`, `title?` | `string` | Button label / header |

**Navigation is lazy** — the picker does NOT fetch the whole tree (it can be huge). It starts from
`GET /categories/roots` and pulls one level on branch click via `GET /categories/{id}` (the `children`
field). A leaf is detected by an empty `children` list (discover-on-click). Each level is cached, so
navigating back via the breadcrumb does not re-fetch.

- **Use-case 1 (implemented):** product-panel filter — the chosen category sets `?category=<id>` on `GET /products`.
- **Use-case 2 (future):** changing a single product's category — same component with `selectionMode="leaf"`.

## Communication

`event-bus` (`@admin/event-bus`) exports a `mitt()` singleton. Events in use:

| Event | Emitter | Listener | Description |
|-------|---------|----------|-------------|
| `productSelected` | products | Shell | Product clicked in the list — carries `{ sku }`. Shell reacts loosely by **navigating to `/products/{sku}`** (see Routing) |
| `productDeleted` | products | Shell | Product deleted |
| `categoryDeleted` | categories | Shell | Category deleted |

## Routing

Routes (paths) are defined **only by the shell** (`react-router-dom` v7) — the single `<BrowserRouter>`
provider. A remote can still **participate** by reading/writing state in the same router's query params
(as `products` does). Requirement: the router must be **shared as a singleton** — and both packages:
`react-router-dom` AND the core `react-router` (which holds the Router contexts). Without the core
singleton, a remote can't see the host's `<BrowserRouter>` (`useSearchParams` throws).

| Path | View |
|------|------|
| `/` | Home (placeholder) |
| `/products` | Product list (`products/App`) |
| `/products?category=<id>&q=<keyword>&page=<n>` | List with filter/search/pagination (state in URL) |
| `/products/:sku` | Product card (`productPage/App`, fetched by SKU) |
| `/categories` | Categories (`categories/App`) |
| `*` | Redirect to `/` |

- Implementation: `shell/src/App.tsx` (`<BrowserRouter>` + `<Routes>`), `shell/src/components/Sidebar.tsx`
  uses `NavLink` (auto-active state; `/products/:sku` highlights "Products" via prefix match).
- **`productSelected` → URL:** the shell subscribes to the event and runs `navigate('/products/{sku}')`,
  storing the list address (with filters) in `location.state.from`. The product list knows no paths —
  it only emits the event (loose coupling).
- **Back from the card** restores the list state: if we came from the list (`state.from`), `navigate(-1)`
  returns to the exact same URL (filter/keyword/page live in the query); deep-link entry falls back to `/products`.
- **Product list state in URL:** `products` uses `useSearchParams` (the shell's shared router) —
  `category`, `q`, `page` are the source of truth; changing the filter/keyword resets `page`. The
  category name for the chip is fetched by `id` via `GET /categories/{id}` (deep-links don't carry the
  name). Standalone (`:3001`) has its own `<BrowserRouter>` in `bootstrap.tsx`; as a remote it uses the host's router.
- Deep-links survive refresh thanks to `historyApiFallback: true` in the shell's dev server.

## Styling

- Each MFE has its own `tailwind.config.js` with the same colors / base styles
- The shell imports `tailwind.css` globally
- Remotes expose styles via Webpack's `MiniCssExtractPlugin`
- No style conflicts — each MFE has scoped styles

## API

Each microfrontend has its own `api.ts` with fetch calls to the backend:

| Endpoint | Method | Microfrontend |
|----------|--------|---------------|
| `/products?category={id}&search=&limit=&offset=` | GET | products (the `category` filter includes subcategories) |
| `/products/by-sku/{sku}` | GET | product-page (product card by SKU) |
| `/products/{id}` | GET | (by numeric ID, unused by the frontend) |
| `/categories` | GET | categories |
| `/categories/tree` | GET | categories |
| `/categories/roots` | GET | category-picker (starting level, lazy navigation) |
| `/categories/{id}` | GET | categories, category-picker (drill down one level) |

The shell **does not know** the endpoints — microfrontends talk to the backend on their own.

## Backend

The backend lives in `@backend/` and is a FastAPI app with:

- SQLite database
- JSON-driven seeding (`products.json`, `categories.json`)
- CORS open (`allow_origins=["*"]`)
- Endpoints: `/products`, `/products/by-sku/{sku}`, `/products/{id}`, `/categories`, `/categories/tree`, `/categories/roots`, `/categories/{id}`, `/health`

## Code Conventions

- TypeScript strict mode
- Each `.tsx` file has a matching component in the same directory
- API clients in `lib/api.ts`
- No comments in code (unless necessary)
- Styling via Tailwind utility classes, no custom CSS (unless necessary)
