# SafeTrust Architecture Contract

This document defines the architectural layers, boundaries, and import contracts for `frontend-SafeTrust`. ESLint enforces these boundaries via `eslint.config.mjs` to keep the codebase maintainable, secure, and predictable.

Each section is labelled **Current**, **Demo**, or **Planned**. Planned items are targets, not rules the build enforces yet.

---

## Test and Mock Conventions (Current)

- Keep unit and integration tests co-located with their subjects as `*.test.ts` or `*.test.tsx` files under `src/`. Do not add `__tests__` directories.
- Keep demo data and fixtures with their features. Keep shared test helpers in `src/test/`.
- The test runner is Jest (`npx jest`).
- Use `renderWithProviders` from `src/test/render.tsx` when tests need the React Query and theme providers or seeded authentication state.
- Keep browser end-to-end tests in `e2e/` as Playwright specs, fixtures, and support files.

> **Basis:** the Jest command in `src/features/escrow/README.md`, at commit `ad2f38d7`.
> **Not yet verified:** `src/test/render.tsx` and `e2e/` exist. Check both before merge.

---

## 1. Directory Structure & Layer Map (Current, with Planned items marked)

```text
src/
├── app/                 Current. Page, layout, loading, error and route files, plus API route
│                        handlers under src/app/api/. Keep server secrets out of page files.
├── features/            Current. One folder per business domain (escrow, apartments, and others).
│   └── <feature>/
│       ├── index.ts         ← the public entry point
│       ├── components/      feature UI (client or server)
│       ├── hooks/           feature hooks
│       ├── data/            repository and demo data
│       ├── model/           types, schemas, state machines, pure logic
│       └── *.test.ts(x)     co-located tests
├── components/
│   ├── ui/              Current. shadcn primitives: stateless, no data, no auth, no wallet
│   └── layouts/         Current. App shell: Header, SideBar, DemoBanner, nav config
├── server/              Planned. Server-only code (firebase-admin, SEP-10, secrets).
│                        Today this code lives in src/lib/auth/*-server.ts and src/app/api/.
├── lib/                 Current. Framework-free helpers shared by two or more features
├── providers/           Current. AppProviders and QueryProvider exist. The root layout does not mount them.
│                        Per-route-group composition is Planned.
├── config/              env.ts (client) and env.server.ts (server)
└── types/               Cross-feature types only. Feature types live in features/<x>/model
```

> **Basis:** folder layout from the issue text and the files supplied in review, at commit `ad2f38d7`.
> **Not yet verified:** `src/server/` is absent; `src/lib/auth/*-server.ts` exists; `src/config/env.ts` and `src/config/env.server.ts` exist.

Note on the old path: earlier versions of this document named `components/layout/`. The folder is `components/layouts/`.

---

## 2. Permitted vs Forbidden Import Table (Current, with Planned rows marked)

| Layer | May import | Must NOT import |
| --- | --- | --- |
| `app/` | Everything except another route's private files | — |
| `features/<x>` | `components/ui`, `components/layouts`, `lib`, `config/env`, `types`, **other features only via their `index.ts`** | `app/`, `server/`, another feature's internal modules (`@/features/*/*`) |
| `components/ui` | `lib/utils` only | `features`, `hooks`, `providers`, `data`, `server/` |
| `components/layouts` | `components/ui`, `lib`, `features/*/index.ts` (auth state only) | `server/`, demo data |
| `server/` (Planned) | `config/env.server`, `lib` | Anything client (`"use client"` modules, React) |
| `lib/` | `config`, `types` | `features`, `components`, `server/` |

> **Basis:** the table as written in the previous version, checked against `src/components/layouts/` and `src/features/` in review, at commit `ad2f38d7`.
> **Not yet verified:** the table matches `eslint.config.mjs` rule for rule. Confirm before merge.

---

## 3. Core Decision Rules (Current)

1. **Consolidate by change and runtime.** Code that changes together and needs the same runtime (wallet, escrow SDK, query client) lives in the same feature and mounts under the same route group.
2. **Isolate by trust and blast radius.** Secrets and server SDKs live on the server side, away from client code. Each route group will have its own error boundary (Planned).
3. **One way in.** A feature is consumed through its public `index.ts`. Deep imports into a feature's internal subdirectories (`@/features/<x>/components/...`) are forbidden.

> **Basis:** the feature layout at commit `ad2f38d7`.
> **Not yet verified:** rule 3 is enforced by ESLint. Confirm the rule in `eslint.config.mjs`.

---

## 4. ESLint Boundary Enforcement (Current)

Rules in `eslint.config.mjs` enforce these layer boundaries:

- **Presentational UI isolation** (`src/components/ui/**`): banned from importing `@/features/*`, `@/hooks/*`, `@/providers/*`, `@/server/*`, `@/lib/mockData*`, `@/lib/demo*`.
- **Feature encapsulation** (`src/features/**`): banned from deep imports into other features (`@/features/*/*`).
- **Shared library purity** (`src/lib/**`): banned from depending on `@/features/*`, `@/components/*`, or `@/server/*`.
- **Module escape prevention** (`src/**`): relative imports that escape module boundaries (`../../../*`) are banned in favor of the `@/` path alias.

The `@/server/*` patterns apply once `src/server/` exists (Planned).

> **Basis:** the rule descriptions in the previous version of this document.
> **Not yet verified:** each pattern above is present in `eslint.config.mjs` at commit `ad2f38d7`. Check each one before merge.

---

## 5. Route Groups & Runtime Profiles (Planned)

Route groups are **Planned**. `src/app/` has no route groups today. The table below is the target design, not the current state.

The target segments the application by runtime profile, so providers and heavy dependencies (such as `@creit.tech/stellar-wallets-kit` and `@trustless-work/escrow`) mount once per group layout rather than per page or globally at root.

### Target group profiles

| Group | Routes | Providers mounted | Target first-load JS |
| --- | --- | --- | --- |
| `(marketing)` | `/`, `/rent`, `/rent/[id]`, `/hotels`, `/hotels/[id]`, `/hotels/search`, `/guest/suggestions` | `QueryProvider` only (RSC-first; no auth listener, no wallet kit) | ≤ 180 kB |
| `(auth)` | `/login`, `/register`, `/forgot-password`, `/reset-password`, `/verify-email` | `AuthProvider` (centered auth shell; wallet login loads kit lazily on click) | ≤ 200 kB |
| `(app)` | `/dashboard/**` | `AuthProvider` + `QueryProvider` + sidebar shell | ≤ 260 kB |
| `(checkout)` | `/rent/[id]/escrow/**`, `/hotels/[id]/book`, `/room`, `/bookings/**` | `AuthProvider` + `QueryProvider` + `WalletProviderScoped` + `EscrowProviders` | ≤ 350 kB |

- **Root `src/app/layout.tsx`** (Current): renders the theme provider, `DemoBanner`, page content, and `Toaster`. It does not mount `AppProviders`. Target: no data, auth, or wallet providers in the root layout. See [Skeleton Architecture § Provider tree](SKELETON_ARCHITECTURE.md#provider-tree-current).
- **Provider Rule** (Planned): pages do not mount data, auth, wallet, or feature providers. Shell providers such as `ThemeProvider` and `Toaster` stay in the root layout. Check with `grep -rn "Provider" src/app --include=page.tsx`, which should return no mounts.

> **Basis:** the route-group table in the previous version of this document, and the root layout as supplied in review.
> **Not yet verified:** whether any route-group folders already exist under `src/app/`. Run `find src/app -type d -name "(*)"`. If any exist, change this heading to Current and move those groups out of Planned.
