# Admin Panel — Microfrontend E-commerce

Minimalistyczny panel administracyjny platformy ecommerce, zbudowany w architekturze Microfrontends z wykorzystaniem **Module Federation 2.0**.

## Architektura

```
admin/
├── apps/                           # KOD — kazdy podfolder to samodzielny komponent
│   │                               #       (wlasny package.json + node_modules), docelowo osobne repo
│   ├── shell/                      # Host — shell glowny z sidebar menu
│   ├── products/                   # Remote — LISTA produktow (konsumuje category-picker)
│   ├── product-page/               # Remote — KARTA pojedynczego produktu (./App, fetch po SKU)
│   ├── categories/                 # Remote — microfrontend kategorii
│   ├── category-picker/            # Remote — reuzywalny picker kategorii (./CategoryPicker)
│   └── event-bus/                  # Wspolny kontrakt — mitt event bus (shared singleton)
├── docs/                           # DOKUMENTACJA (repo-split.md, ...)
└── AGENTS.md
```

Nie ma juz nadrzednego workspace'u npm — kazdy komponent w `apps/` instaluje i buduje sie samodzielnie
(`cd apps/<x> && npm install`). `event-bus` jest konsumowany przez `file:../event-bus`.

| Rola | Aplikacja | Opis |
|------|-----------|------|
| Host | `apps/shell` | Gowna aplikacja, sidebar, routing, renderuje RemoteModules |
| Remote | `apps/products` | Lista produktow — paginacja + wyszukiwanie + filtr kategorii (emituje `productSelected`) |
| Remote | `apps/product-page` | Karta pojedynczego produktu — pobiera produkt po SKU z backendu |
| Remote | `apps/categories` | Panel kategorii — drzewo kategorii + szczegoly |
| Remote | `apps/category-picker` | Reuzywalny komponent wyboru kategorii (`./CategoryPicker`), konsumowany przez inne MFE |
| Shared | `apps/event-bus` | Singleton mitt (pakiet `@admin/event-bus`) do komunikacji miedzy microfrontendami |

## Technologie

- **React 19** z TypeScript
- **Webpack 5 + `@module-federation/enhanced`** — Module Federation 2.0 (react/react-dom jako `singleton`, bez duplikacji)
- **Tailwind CSS** — kazdy microfrontend ma wlasciwy config, shell ładuje style globalne
- **mitt** — event bus w `event-bus`
- **react-router-dom v7** — routing po URL; sciezki w shellu, query params tez w `products`; `react-router(-dom)` jako MF `singleton`
- **Vite** nie jest wykorzystywany — pelny webpack 5 z MF pluginem

## Module Federation 2.0

Plugin pochodzi z `@module-federation/enhanced` (MF 2.0), nie z wbudowanego `webpack.container`:

```ts
import { ModuleFederationPlugin } from "@module-federation/enhanced/webpack";
```

`dts: false` — typy remote'ow deklarujemy recznie w `shell/src/types.d.ts`, bez automatycznej generacji.

### Host (`shell`)

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
    // Bez singletonu kazdy MFE ma wlasna instancje mitt -> zdarzenia nie przechodza
    "@admin/event-bus": { singleton: true, requiredVersion: false },
    // Jeden router na cala aplikacje (jedna historia/kontekst)
    "react-router-dom": { singleton: true, requiredVersion: deps["react-router-dom"] },
  },
  dts: false,
})
```

### Remotes (`products`, `categories`)

```ts
new ModuleFederationPlugin({
  name: "products", // lub "categories"
  filename: "remoteEntry.js",
  exposes: {
    "./App": "./src/App",
  },
  shared: {
    react: { singleton: true, requiredVersion: deps.react },
    "react-dom": { singleton: true, requiredVersion: deps["react-dom"] },
    // tylko w remote'ach uzywajacych szyny (products, categories)
    "@admin/event-bus": { singleton: true, requiredVersion: false },
  },
  dts: false,
})
```

### Konsumpcja remote'ow

Shell laduje remote'y standardowo przez `React.lazy` + dynamiczny `import` (bez globali na `window`). `RemoteModule` mapuje aktywna karte na leniwy komponent i opakowuje go w `ErrorBoundary` (fallback gdy remote jest offline); `Suspense` jest w `App.tsx`:

```tsx
const remotes = {
  products: React.lazy(() => import("products/App")),
  categories: React.lazy(() => import("categories/App")),
};
```

### Async boundary (bootstrap)

`shared` z `singleton` wymaga asynchronicznej inicjalizacji share-scope, dlatego entry kazdej aplikacji jest rozbity:

- `src/main.tsx` (shell) / `src/index.ts` (remote) — tylko `import("./bootstrap")`
- `src/bootstrap.tsx` — wlasciwe `ReactDOM.createRoot(...).render(<App />)`

Remote'y montuja sie do `#root` w trybie standalone (`:3001`, `:3002`); jako remote eksponuja wylacznie `./App`. Import CSS jest w `App.tsx` (a nie w bootstrapie), zeby style podrozowaly z eksponowanym modulem i dzialaly rowniez u hosta.

### Category Picker (reuzywalny remote)

`category-picker` (port `:3003`) eksponuje **`./CategoryPicker`** — jeden komponent przeznaczony do wielu use-case'ow. Konsumowany jest jak kazdy remote, ale to **komponent z propsami** (nie cala strona), wiec komunikacja idzie przez callback `onSelect`, a nie event-bus:

```ts
// products/webpack.config.mjs
remotes: { categoryPicker: "categoryPicker@http://localhost:3003/remoteEntry.js" }
```

```tsx
const CategoryPicker = React.lazy(() => import("categoryPicker/CategoryPicker"));
// opakowany w ErrorBoundary + Suspense (fallback gdy picker offline)
<CategoryPicker selectionMode="any" onSelect={(sel) => ...} onCancel={...} />
```

Propsy (typy zadeklarowane recznie w `products/src/types.d.ts`, `dts:false`):

| Prop | Typ | Opis |
|------|-----|------|
| `onSelect` | `(sel: { id; name; path }) => void` | Zwraca wybrana kategorie + sciezke breadcrumb |
| `onCancel?` | `() => void` | Renderuje przycisk „Anuluj" |
| `selectionMode?` | `"leaf" \| "any"` | `leaf` — wybieralne tylko liscie (np. zmiana kategorii produktu); `any` — takze biezaca galaz (filtr; backend i tak filtruje wraz z podkategoriami). Domyslnie `leaf` |
| `confirmLabel?`, `title?` | `string` | Etykieta przycisku / naglowek |

**Nawigacja jest leniwa** — picker NIE pobiera calego drzewa (moze byc ogromne). Startuje od `GET /categories/roots`, a po kliknieciu w galaz dociaga jeden poziom przez `GET /categories/{id}` (pole `children`). Lisc rozpoznawany jest po pustej liscie `children` (discover-on-click). Kazdy poziom jest cache'owany, wiec cofanie po breadcrumb nie pobiera ponownie.

**Use-case 1 (zaimplementowany):** filtr na panelu produktow — wybrana kategoria ustawia `?category=<id>` w `GET /products`.
**Use-case 2 (przyszly):** zmiana kategorii pojedynczego produktu — ten sam komponent z `selectionMode="leaf"`.

## Komunikacja

`event-bus` (pakiet `@admin/event-bus`) eksportuje singleton `mitt()`. Uzywane eventy:

| Event | Kto wysyla | Kto odbiera | Opis |
|-------|-----------|-------------|------|
| `productSelected` | products | Shell | Klikniecie produktu na liscie — niesie `{ sku }`. Shell luzno reaguje, **nawigujac na `/products/{sku}`** (patrz Routing) |
| `productDeleted` | products | Shell | Usunieto produkt |
| `categoryDeleted` | categories | Shell | Usunieto kategorie |

## Routing

Trasy (sciezki) definiuje **wylacznie shell** (`react-router-dom` v7) — to jedyny provider `<BrowserRouter>`.
Remote moze jednak **uczestniczyc** w routingu, czytajac/zapisujac stan w query params tego samego routera
(tak robi `products`). Warunek: router musi byc **wspoldzielony jako singleton** w `shared` — i to oba
pakiety: `react-router-dom` ORAZ rdzen `react-router` (to on trzyma konteksty Routera). Bez singletonu
rdzenia remote nie widzi `<BrowserRouter>` hosta (`useSearchParams` rzuca bledem).

| Sciezka | Widok |
|---------|-------|
| `/` | Strona glowna (placeholder) |
| `/products` | Lista produktow (`products/App`) |
| `/products?category=<id>&q=<keyword>&page=<n>` | Lista z filtrem/wyszukiwaniem/paginacja (stan w URL) |
| `/products/:sku` | Karta produktu (`productPage/App`, pobiera po SKU) |
| `/categories` | Kategorie (`categories/App`) |
| `*` | Redirect na `/` |

- Implementacja: `shell/src/App.tsx` (`<BrowserRouter>` + `<Routes>`), `shell/src/components/Sidebar.tsx`
  uzywa `NavLink` (auto-aktywny stan; `/products/:sku` podswietla „Produkty" dzieki dopasowaniu prefiksu).
- **`productSelected` → URL:** shell subskrybuje event i robi `navigate('/products/{sku}')`, zapisujac adres
  listy (z filtrami) w `location.state.from`. Lista produktow nie zna sciezek — emituje tylko event (luzne spiecie).
- **Powrot na karcie** odtwarza stan listy: jesli przyszlismy z listy (`state.from`), `navigate(-1)` wraca na
  dokladnie ten sam URL (filtr/keyword/page sa w query, wiec wracaja same); przy wejsciu z deep-linku fallback na `/products`.
- **Stan listy produktow w URL:** `products` uzywa `useSearchParams` (wspoldzielony router shella) — `category`,
  `q`, `page` sa zrodlem prawdy; zmiana filtra/keywordu resetuje `page`. Nazwa kategorii do chipa pobierana
  jest po `id` z `GET /categories/{id}` (deep-link nie zna nazwy). Standalone (`:3001`) ma wlasny
  `<BrowserRouter>` w `bootstrap.tsx`; jako remote uzywa routera hosta.
- Deep-linki dzialaja po odswiezeniu dzieki `historyApiFallback: true` w devServerze shella.

## Styling

- Kazdy microfrontend ma wlasny `tailwind.config.js` z tymi samymi kolorami/base styles
- Shell importuje `tailwind.css` globalnie
- Remote mody udostepniaja style przez webpack `MiniCssExtractPlugin`
- Brak konfliktow stylow — kazdy MFE ma scoped styles

## API

Kazdy microfrontend ma wlasciwy `api.ts` z fetch do backendu:

| Endpoint | Method | Microfrontend |
|----------|--------|---------------|
| `/products?category={id}&search=&limit=&offset=` | GET | products (filtr `category` obejmuje podkategorie) |
| `/products/by-sku/{sku}` | GET | product-page (karta produktu po SKU) |
| `/products/{id}` | GET | (po numerycznym ID, nieuzywane przez front) |
| `/categories` | GET | categories |
| `/categories/tree` | GET | categories |
| `/categories/roots` | GET | category-picker (poziom startowy, leniwa nawigacja) |
| `/categories/{id}` | GET | categories, category-picker (drazenie o jeden poziom) |

Shell **nie zna** endpointow — microfrontendy komunikuja sie z backendem samodzielnie.

## Backend

Backend znajduje sie w `@backend/` i jest to FastAPI z:
- SQLite baza danych
- JSON-driven seeding (products.json, categories.json)
- CORS wylaczony (`allow_origins=["*"]`)
- Endpointy: `/products`, `/products/by-sku/{sku}`, `/products/{id}`, `/categories`, `/categories/tree`, `/categories/roots`, `/categories/{id}`, `/health`

## Uruchamianie

Kazdy komponent jest samodzielny — instalujesz i uruchamiasz go osobno (jak osobne repo).

```bash
# Backend (w osobnym terminalu)
cd ../backend && ./run.sh start

# Kazdy komponent w osobnym terminalu (przyklad: shell)
cd apps/shell && npm install && npm run dev
# analogicznie: apps/products, apps/categories, apps/category-picker, apps/product-page
```

Do uruchomienia wszystkiego naraz mozesz uzyc wlasnego skryptu/`concurrently` albo cienkiego
meta-repo (patrz `docs/repo-split.md`). Kolejnosc nie ma znaczenia — remote'y lacza sie z shellem w runtime.

Porty:
- `apps/shell` — :3000
- `apps/products` — :3001
- `apps/categories` — :3002
- `apps/category-picker` — :3003
- `apps/product-page` — :3004

## Konwencje kodu

- TypeScript strict mode
- Kazdy plik .tsx ma odpowiadajacy komponent w tym samym katalogu
- API clients w `lib/api.ts`
- Brak komentarzy w kodzie (chyba ze konieczne)
- Puste `__init__.py` nie jest potrzebne (to nie jest Python)
- Style przez Tailwind utility classes, brak custom CSS (chyba ze konieczne)
