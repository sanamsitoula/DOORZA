# Doorza (Spree Commerce 6.0) — Project Description

Cloned from `https://github.com/sanamsitoula/DOORZA.git`. The upstream project is
**Spree Commerce**, an open-source, headless e-commerce platform. Backend is a
Ruby on Rails engine suite; admin/seller UIs are React SPAs; storefront is a
separate Next.js repo. Everything is orchestrated as one **pnpm + Turbo**
monorepo.

## 1. Product Features

- **Catalog**: products, variants, option types/values, categories (hierarchical,
  nested-set), collections (rule-based or manual merchandising groups), product
  types, custom fields, media/digital assets.
- **Orders & Fulfillment**: cart → checkout → payment → fulfillment → returns/
  claims/exchanges, gift cards, store credits, discounts, promotions.
- **Multi-tenant stores**: every domain model is store-scoped (`belongs_to :store`);
  no cross-store data sharing (`spree_multi_store` legacy path removed).
- **Marketplace mode**: sellers, seller payouts, seller requirements, commission
  rates/lines, purchase orders, stock transfers between suppliers.
- **Admin back office**: React SPA (`@spree/dashboard`) covering catalog, orders,
  customers, promotions, taxes, shipping, reporting, settings, imports/exports.
- **Seller panel**: separate React SPA (`@spree/seller-dashboard`) for marketplace
  sellers, built on the same design system.
- **Storefront**: Next.js app (`spree/storefront`, cloned separately, not a
  workspace member) consuming the Store API via `@spree/sdk`.
- **Integrations**: Stripe (payments/connect payouts), EasyPost (shipping rates/
  labels), Meilisearch (search), OpenTelemetry (observability), webhooks.
- **Compliance**: GDPR-style data requests (export/anonymize), consent records,
  audit-friendly API key scopes.

## 2. Code Architecture

Layered around a REST **API v3** that both the Admin SPA and the Store API
consume — no server-rendered admin views remain in 6.0.

```
Client (Dashboard / Seller Panel / Storefront)
        │  HTTPS (publishable key / secret key / JWT)
        ▼
Spree::Api::V3::{Store,Admin,Seller}::*Controller   (spree/api)
        │
        ├─ ability/scope checks (CanCanCan, or API-key scopes)
        ├─ Workflows (app/workflows) — multi-step domain operations
        ▼
Spree::* Active Record models (spree/core)           — business logic, validations
        │
        ▼
PostgreSQL (or MySQL/SQLite)
```

Controllers never touch the DB with raw SQL and never hold business logic —
they authenticate, authorize, permit params, call a workflow or the model, and
serialize the result via Alba serializers.

## 3. Coding Practices (backend, from `CLAUDE.md`)

- Everything namespaced under `Spree::`; Rails + CanCanCan + Ransack + Pagy.
- **Tenant safety**: every controller query goes through `current_store.<assoc>`,
  never `Spree::Model` directly — turns cross-store ID access into a 404 (cheap
  IDOR defense). `accessible_by` alone is not a substitute (that's role, not tenant).
- **Prefixed IDs** everywhere on the wire (`prod_86Rf07xd4z`), never raw integers.
- **Workflows over hand-rolled actions**: a controller that needs custom create/
  update logic declares `create_workflow`/`update_workflow` and keeps the
  inherited `create`/`update` (authorization + rendering stay correct); only
  hand-write the action when the flow genuinely can't fit that shape (see
  `CustomersController` below).
- **Read/write symmetry**: whatever a serializer exposes is exactly what the
  controller permits on write; a renamed column gets a model-level alias, never
  a client-side field-name translation.
- No state machines (removed since 6.0) — statuses are plain string columns
  moved by **Workflows**, declared with `has_status`.
- No foreign-key constraints in migrations; `null: false` on required columns;
  soft delete via `paranoia`/`acts_as_paranoid`.
- Frontend: Vite + TanStack Router/Query, React Hook Form + Zod, shadcn/ui on
  Base UI (not Radix) + Tailwind, Biome for lint/format, i18next for every
  user-visible string, destructive actions always behind `useConfirm()`.

## 4. Folder Structure

```
doorza/
├── spree/                          Ruby engine gems (the backend)
│   ├── core/                       spree_core — models, business logic
│   ├── api/                        spree_api  — Store/Admin/Seller REST controllers, serializers, routes
│   ├── emails/                     spree_emails — transactional email
│   ├── dashboard/                  spree_dashboard — hosts the built React dashboard at /dashboard
│   ├── opentelemetry/              spree_opentelemetry — tracing
│   └── providers/                  spree_easypost, spree_meilisearch, spree_stripe
├── packages/                       TypeScript workspace (the frontend)
│   ├── sdk / sdk-core              Store API TS client (+ shared HTTP layer)
│   ├── admin-sdk                   Admin API TS client
│   ├── seller-sdk                  Seller API TS client
│   ├── dashboard                   @spree/dashboard — admin SPA (routes, hooks, schemas)
│   ├── dashboard-core              registries, providers, plugin API
│   ├── dashboard-ui                design system (headless, Base UI + shadcn)
│   ├── dashboard-starter           thin host app that boots @spree/dashboard
│   ├── seller-dashboard(-starter)  marketplace seller panel, mirrors dashboard split
│   └── cli / create-spree-app      project scaffolding & Docker-based CLI
├── server/                         .gitignored — spree-starter Rails app clone (per worktree)
├── storefront/                     .gitignored — separate Next.js repo clone (own .git)
├── docs/                           Mintlify docs site (spreecommerce.org/docs)
├── scripts/                        dev/server/worktree automation
├── CLAUDE.md                       full contributor rulebook (this file summarizes it)
└── package.json / pnpm-workspace.yaml / turbo.json
```

Inside every `spree/*` gem: `app/models/spree/`, `app/controllers/spree/`,
`app/serializers/spree/`, `app/services/`, `app/jobs/`, `app/workflows/`,
`config/routes.rb`, `spec/`.

## 5. Four CRUD Modules, Explained

All four live in `spree/api`, mounted under `Spree::Api::V3::Admin`
(`/api/v3/admin/...`), and share one inheritance chain:

```
Spree::Api::V3::ResourceController        (spree/api/app/controllers/spree/api/v3/resource_controller.rb)
        │  index / show / create / update / destroy — generic, workflow-aware
        ▼
Spree::Api::V3::Admin::ResourceController (spree/api/app/controllers/spree/api/v3/admin/resource_controller.rb)
        │  + CanCanCan authorization, admin auth, store context
        ▼
<Resource>Controller                      (one per module — only overrides what differs)
```

The base `ResourceController#create`/`#update`/`#destroy`/`#index`/`#show`
implement the actual HTTP verbs once; a subclass typically only defines
`model_class`, `serializer_class`, and `permitted_params`.

### 5.1 Products — full custom workflow CRUD

| Piece | Path |
|---|---|
| Routes | `spree/api/config/routes.rb` (`resources :products`, line ~331) |
| Controller | `spree/api/app/controllers/spree/api/v3/admin/products_controller.rb` |
| Model | `spree/core/app/models/spree/product.rb` |
| Serializer | `spree/api/app/serializers/spree/api/v3/admin/product_serializer.rb` |

`GET/POST /api/v3/admin/products`, `GET/PATCH/DELETE /api/v3/admin/products/:id`,
plus member actions (`clone`, `approve`, `reject`) and bulk collection actions
(`bulk_status_update`, `bulk_add_to_categories`, `bulk_destroy`, ...). Create/
update are routed through `Spree.product_create_workflow` /
`Spree.product_update_workflow`; destroy runs `Spree.product_destroy_workflow`
because deleting a product has side effects (variants, stock, search index).
Nested resources: `variants`, `media`, `digital_assets`.

### 5.2 Categories — inherited CRUD + one custom member action

| Piece | Path |
|---|---|
| Routes | `spree/api/config/routes.rb` (`resources :categories`, line ~381) |
| Controller | `spree/api/app/controllers/spree/api/v3/admin/categories_controller.rb` |
| Model | `spree/core/app/models/spree/category.rb` |
| Serializer | `spree/api/app/serializers/spree/api/v3/admin/category_serializer.rb` |

`GET/POST /api/v3/admin/categories`, `GET/PATCH/DELETE .../:id`, plus
`PATCH .../:id/reposition` (moves a node in the nested-set tree —
`acts_as_nested_set`). No `create_workflow`/`update_workflow` override, so
create/update save the AR record directly; `scope` is narrowed to `.manual`
(rule-based taxons are excluded from this API).

### 5.3 Customers — hand-written CRUD (workflow doesn't fit)

| Piece | Path |
|---|---|
| Routes | `spree/api/config/routes.rb` (`resources :customers`, line ~528) |
| Controller | `spree/api/app/controllers/spree/api/v3/admin/customers_controller.rb` |
| Model | `spree/core/app/models/spree/customer.rb` |
| Serializer | `spree/api/app/serializers/spree/api/v3/admin/customer_serializer.rb` |

Overrides `create`/`update`/`destroy` explicitly (admin-created customers can
skip password; a `consent_source` flag must be stamped on update; destroy must
rescue `DestroyWithOrdersError`). Plus GDPR actions `export` and `anonymize`,
and bulk group membership actions. This is the textbook case for *not* using
`create_workflow`/`update_workflow` — the per-field side effects don't fit that
shape, so the actions are written by hand instead of forced into it.

### 5.4 Option Types — pure inherited CRUD (minimal example)

| Piece | Path |
|---|---|
| Routes | `spree/api/config/routes.rb` (`resources :option_types`, line ~404) |
| Controller | `spree/api/app/controllers/spree/api/v3/admin/option_types_controller.rb` |
| Model | `spree/core/app/models/spree/option_type.rb` |
| Serializer | `spree/api/app/serializers/spree/api/v3/admin/option_type_serializer.rb` |

The cleanest example of the pattern — ~25 lines total. No custom actions, no
workflow, no overridden `create`/`update`/`destroy`: `model_class`,
`serializer_class`, `scope_includes`, and `permitted_params` are the only
overrides, and the base class does the rest (including nested
`option_values` written inline through `permitted_params`).

## 6. Install & Run — what was actually done in this environment

Environment has: Node 24, pnpm 11, Docker. No local Ruby, no local PostgreSQL,
so the Rails backend runs via the Docker edge path.

```bash
pnpm install         # workspace deps — 711 packages
pnpm build           # turbo build, all 15 workspace packages — 12/12 green
pnpm server:create   # clones spree-starter (branch 6-0-dev) into ./server
```

**Backend is live**: `docker compose -f server/docker-compose.dev.yml -f
scripts/docker-compose.edge.yml up web` — Postgres 18, mailpit, and the Rails
app all running; migrations ran, DB seeded, Puma listening on
`http://localhost:3000` (`GET /` → 200, `GET /api/v3/store/products` → 401 as
expected without a publishable key).

Getting there on Windows needed three fixes beyond the documented (Mac/Linux)
flow — worth knowing if this gets re-run:

1. **Docker Desktop `httpReadSeeker: EOF` on every image pull.** Caused by
   the experimental containerd image-store snapshotter
   (`UseContainerdSnapshotter: true` in `%APPDATA%\Docker\settings-store.json`)
   — its lazy-pull streaming has no retry and dies on any connection blip.
   Fix: set it to `false`, restart Docker Desktop. Confirmed by `Storage
   Driver: overlay2` in `docker info` afterward; pulls then auto-retry and
   succeed.
2. **`env: 'ruby\r': No such file or directory` in the image build.** Git
   checked `server/bin/*` out with CRLF line endings (Windows default), which
   breaks the `#!/usr/bin/env ruby` shebang inside the Linux container. Fix:
   strip `\r` from every file in `server/bin/`.
3. **`mount denied: ... too many colons`.** `scripts/docker-compose.edge.yml`
   mirror-mounts the monorepo at `${SPREE_PATH}:${SPREE_PATH}` — the same
   path on host and in-container, which is how the Gemfile's
   `path "#{ENV['SPREE_PATH']}/spree"` resolves identically both sides. That
   assumes a POSIX-shaped path (true on Mac/Linux); a Windows path
   (`C:/claude_projects/doorza`) used as *both* source and target overflows
   Docker's colon-based volume-string parser, and a Windows-style path is
   also invalid as a Linux container path anyway. Fix: use the WSL-style form
   for `SPREE_PATH` so it's valid on both sides —
   `MSYS_NO_PATHCONV=1 SPREE_PATH=/c/claude_projects/doorza docker compose ...`.

```bash
# Full command that worked:
MSYS_NO_PATHCONV=1 SPREE_PATH=/c/claude_projects/doorza \
  docker compose -f server/docker-compose.dev.yml -f scripts/docker-compose.edge.yml \
  up --force-recreate --remove-orphans web
```

**Admin dashboard is also live**: `pnpm dashboard:dev` (its underlying
`package.json` script) fails on Windows for a fourth, separate reason — pnpm
runs `package.json` scripts through `cmd.exe`, whose escape character is `^`,
which corrupts turbo's dependency-only filter syntax
(`--filter='@spree/dashboard-starter^...'` arrives with the `^` stripped, so
turbo can't find the package). Fix: run the two commands the script chains
directly from bash instead of through the pnpm script wrapper:

```bash
pnpm exec turbo build --filter='@spree/dashboard-starter^...'
pnpm --dir packages/dashboard-starter dev --port 5173 --strictPort
```

Vite came up in ~12s, `GET http://localhost:5173/` → 200.

**To finish setup**: open the one-time link the backend boot log printed
(`http://localhost:5173/setup?token=...`) to create the first admin account.
Admin login afterward: `spree@example.com` / `spree123`.

Summary of the four Windows-specific fixes needed, none of which are bugs in
the project — the documented flow is Mac/Linux-first:

| # | Symptom | Cause | Fix |
|---|---|---|---|
| 1 | `httpReadSeeker: EOF` on every `docker pull` | Docker Desktop's containerd snapshotter has no retry on dropped streams | `UseContainerdSnapshotter: false` in Docker Desktop settings, restart |
| 2 | `env: 'ruby\r': No such file` in image build | `server/bin/*` checked out with CRLF | strip `\r` from `server/bin/*` |
| 3 | `mount denied: too many colons` | Windows path used as both source+target in `${SPREE_PATH}:${SPREE_PATH}` mirror-mount | use WSL-style `SPREE_PATH=/c/...` with `MSYS_NO_PATHCONV=1` |
| 4 | turbo `--filter` loses its `^` | pnpm scripts run via `cmd.exe`, whose escape char is `^` | run the script's commands directly from bash, skip the pnpm-script wrapper |

## 7. Knowledge Graph

A `graphify` knowledge graph was built over `spree/` (backend engines only —
the full repo is 6,891 files / 6.3M words, over graphify's auto-run threshold;
`packages/` and `docs/` were excluded from this pass). Outputs in
`graphify-out/`: `graph.html` (interactive), `GRAPH_REPORT.md` (audit trail),
`graph.json` (raw graph — nodes/edges from AST extraction over 3,328 Ruby
files plus semantic extraction over the 13 real README/manifest docs).

## 8. Suraj Mobiles Shop Customization

Configured for **Suraj Mobiles** (FB: `https://www.facebook.com/profile.php?id=61592995824802`), a modern gadget, mobile, eyewear, and fashion watch retail brand.

### Product Catalog:
- **Mobiles & Smartphones**: Flagship 5G Smartphone Pro 256GB (Titanium Black, Natural Titanium).
- **Mobile Accessories**: 
  - Pro Active Noise Cancelling TWS Earbuds with Spatial Audio (Midnight Black, Glacier White).
  - 65W GaN Dual USB-C Fast Wall Charger.
- **Model Sunglasses & Eyewear**:
  - Model Polarized Aviator Sunglasses UV400 (Gold/Green, Matte Black).
  - Stylish Wayfarer Sunglasses UV400 (Glossy Black, Tortoise Shell).
- **Stylish Watches & Smartwatches**:
  - AMOLED Calling Sports Smartwatch (Obsidian Black, Titanium Silver).
  - Classic Chronograph Stainless Steel Quartz Watch (Royal Blue, Onyx Black).

