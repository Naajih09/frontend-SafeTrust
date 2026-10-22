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

> **Verified against:** `src/features/escrow/README.md` (Jest), `src/test/`, `e2e/` at `ad2f38d7`. Status: Jest and co-location are Current. Confirm `renderWithProviders` and `e2e/` exist before relying on them.

---

## 1. Directory Structure & Layer Map (Current, with Planned items marked)

```text
src/
├── app/                 Current. Routing only: page, layout, loading, error and route files,
│                        metadata, params, auth gates.
├── features/            Current. One folder per business domain: escrow, apartments, and others.
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
├── providers/           Current. AppProviders mounted in the root layout. Per-route-group
│                        composition is Planned.
├── config/              env.ts (client) and env.server.ts (server)
└── types/               Cross-feature types only. Feature types live in features/<x>/model
```

> **Verified against:** `src/features/`, `src/components/layouts/`, `src/lib/auth/`, `src/app/api/` at `ad2f38d7`. The `components/layouts/` path is corrected from `components/layout/`. `src/server/` does not exist yet.

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

> **Verified against:** `eslint.config.mjs`, `src/components/layouts/`, `src/features/` at `ad2f38d7`. Status: Current. The `server/` row is Planned until `src/server/` exists.

---

## 3. Core Decision Rules (Current)

1. **Consolidate by change and runtime.** Code that changes together and needs the same runtime (wallet, escrow SDK, query client) lives in the same feature and mounts under the same route group.
2. **Isolate by trust and blast radius.** Secrets and server SDKs live on the server side, away from client code. Each route group will have its own error boundary (Planned).
3. **One way in.** A feature is consumed through its public `index.ts`. Deep imports into a feature's internal subdirectories (`@/features/<x>/components/...`) are forbidden.

> **Verified against:** `src/features/` layout at `ad2f38d7`. Status: Current for rules 1 and 3, Planned for the per-group error boundary in rule 2.

---

## 4. ESLint Boundary Enforcement (Current, with Planned rules marked)

Rules in `eslint.config.mjs` enforce these layer boundaries:

- **Presentational UI isolation** (`src/components/ui/**`): banned from importing `@/features/*`, `@/hooks/*`, `@/providers/*`, `@/server/*`, `@/lib/mockData*`, `@/lib/demo*`.
- **Feature encapsulation** (`src/features/**`): banned from deep imports into other features (`@/features/*/*`).
- **Shared library purity** (`src/lib/**`): banned from depending on `@/features/*`, `@/components/*`, or `@/server/*`.
- **Module escape prevention** (`src/**`): relative imports that escape module boundaries (`../../../*`) are banned in favor of the `@/` path alias.

The `@/server/*` patterns apply once `src/server/` exists (Planned).

> **Verified against:** `eslint.config.mjs` at `ad2f38d7`. Confirm each pattern is present in the file before relying on it.

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

- **Root `src/app/layout.tsx`** (Current): HTML, body fonts, `globals.css`, `ThemeProvider`, `DemoBanner`, and `Toaster`. Target: no data, auth, or wallet providers in the root layout. Today `AppProviders` is mounted there. See [Skeleton Architecture § Provider tree](SKELETON_ARCHITECTURE.md#provider-tree-current).
- **Provider rule** (Planned): pages never mount providers. This is checked with `grep -rn "Provider" src/app --include=page.tsx`, which must return no mounts. Route groups do not exist yet, so this rule is a target.

> **Verified against:** `src/app/` (no route groups) and `src/app/layout.tsx` at `ad2f38d7`. Status: Planned for the table and provider rule; Current for the root layout.
