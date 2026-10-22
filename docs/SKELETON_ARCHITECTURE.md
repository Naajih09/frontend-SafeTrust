# Skeleton Architecture — frontend-SafeTrust

> For the comprehensive layer contracts, folder structure, and ESLint import boundaries, see [Architecture Contract](ARCHITECTURE.md).

Each section is labelled with its status:

- **Current**: exists in this repo today.
- **Demo**: fixtures or mock code that stands in for a real service.
- **Planned**: not built yet.

## Two-repo strategy (Current, Demo, Planned)

| | Current in this repo | Demo data | Planned (backend-SafeTrust / Hasura) |
| --- | --- | --- | --- |
| **Listings** | Apartment listings use a live Hasura adapter in `src/features/apartments` | Fixture listings in `src/lib/mockData` for screens not yet wired | Hasura as the source for all listings |
| **Escrow** | Booking checkout deploys and funds real escrows with `@trustless-work/escrow` on Stellar. The network comes from `NEXT_PUBLIC_TRUSTLESS_NETWORK`, which defaults to `testnet` | Demo flows read escrow status through `useEscrowSubscription` | Escrow state written only by backend-SafeTrust and read through Hasura subscriptions |
| **Backend calls** | Some screens call the backend through `BACKEND_URL` | Other screens return fixture responses | backend-SafeTrust as the single write authority |
| **Webhooks** | None. This repo exposes no webhook endpoints | Not applicable | Trustless Work webhooks deliver to backend-SafeTrust (`trustless_work_webhook_events`) |

The UI observes escrow changes through `useEscrowSubscription`. It is a mock in this repo and a Hasura subscription in the integrated app. Only the hook changes when a component is promoted.

> **Basis:** `src/features/escrow/README.md`, `package.json`, and the files supplied in review, at commit `ad2f38d7`.
> **Not yet verified:** the apartments live adapter at `src/features/apartments`; the `BACKEND_URL` usage; the defaults in `NEXT_PUBLIC_TRUSTLESS_NETWORK`.

## Data layer (Demo)

```text
src/lib/mockData/                  ← source of truth for stub data
src/hooks/useApartments.ts         ← mock hook (Apollo { data, loading, error } shape)
src/hooks/useEscrowSubscription.ts ← mock hook ({ escrow, loading, error } shape)
```

Query hooks return `{ data, loading, error }`, matching Apollo's `useQuery` return shape. Subscription hooks return resource-specific contracts such as `{ escrow, loading, error }`. When a component is promoted to the integrated app, only the hook import changes. The JSX stays the same.

> **Basis:** the hook names and shapes in the previous version of this document.
> **Not yet verified:** the three paths above exist at commit `ad2f38d7`.

## Repository pattern (Current)

This pattern replaces the older practice of faking mutation success with a timer. Each feature puts its data access behind a repository interface with two adapters:

- **Demo adapter.** Returns fixture data and marks its result as demo. It saves nothing, and the UI says so, for example "Demo: not saved".
- **Live adapter.** Calls the real service. It resolves only after the service confirms the write.

The UI calls the repository, never an adapter directly. It shows success only after the repository's promise resolves with confirmation. It never shows success on a timer.

Illustrative shape. The real code is in `src/features/apartments`:

```ts
export interface ListingRepository {
  createListing(input: NewListing): Promise<Listing>;
}
```

> **Basis:** the pattern as described in the issue.
> **Not yet verified:** `src/features/apartments` contains a repository interface and a live adapter, and the interface name and method above match it.

## Provider tree (Current)

```text
src/providers/
├── AppProviders.tsx           ← root client providers
├── QueryProvider.tsx          ← single QueryClient (+ Devtools in dev)
└── EscrowProviders.tsx        ← Trustless Work + EscrowProvider scope
```

`src/app/layout.tsx` does not mount `AppProviders`. It renders the theme provider, demo banner, page content, and toaster. `EscrowProviders` is mounted only by components that use Trustless Work escrow features, such as `BookingEscrowWrapper` and `HotelMilestoneActions`. Composing providers per route group is **Planned**. See [Architecture Contract § Route Groups](ARCHITECTURE.md#5-route-groups--runtime-profiles-planned).

> **Basis:** the root layout and `src/providers/` as supplied in review, at commit `ad2f38d7`.
> **Not yet verified:** the components named as mounting `EscrowProviders`.

## Auth store (Current)

`src/core/store/data/index.ts` owns the single Zustand wallet store. It starts with an empty address, name, adapter ID, and token. Wallet metadata (`address`, `name`, `walletId`) is persisted under `safetrust-wallet`. Tokens, actions, identity claims, and roles are never written to or restored from that storage. The adapter ID lets xBull or LOBSTR remain the selected signer after a refresh.

`src/hooks/useWallet.ts` exposes `address`, `name`, `isConnected`, `connect`, `disconnect`, and `sign`. The wallet kit loads lazily in the browser from `src/lib/stellar/wallet-kit.ts` and survives hot reload. Its network comes from `NEXT_PUBLIC_TRUSTLESS_NETWORK` (`testnet` by default), which the escrow API and explorer configuration share. `signXdr` is the signing boundary.

Wallet metadata is a connection hint, not authorization. Login uses the server-verified SEP-10 challenge and a Firebase session. A cached address cannot redirect through login or grant dashboard access. The session listener clears expired tokens without erasing wallet metadata. Explicit logout clears both. Disconnecting a wallet clears its metadata and leaves any Firebase session in place.

> **Basis:** the store and hook descriptions in the previous version of this document.
> **Not yet verified:** the three file paths and the storage key `safetrust-wallet` in the code.

## Component promotion path (Demo to live)

When a UI improvement is ready to be wired to real data:

1. Copy the component file verbatim.
2. Replace the mock hook import with the live hook or Apollo hook.
3. Replace the demo adapter with the live adapter, or the stub submit with the real mutation.
4. The JSX body does not change.

This is the **slice pattern**. frontend-SafeTrust is the design and UX source of truth. The integrated app is the data integration layer.

> **Basis:** the promotion steps in the previous version of this document.
> **Not yet verified:** the promotion steps match how `src/features/apartments` was actually built.

## Dependency rules (Current)

| Package | Allowed | Notes |
| --- | --- | --- |
| `lucide-react` | ✅ | Icons. Use this, not `react-icons` |
| `sonner` | ✅ | Toast notifications |
| `zustand` | ✅ | Auth store |
| `date-fns` | ✅ | Date formatting in messages |
| `firebase` | ✅ | Client SDK only (login and register forms) |
| `@trustless-work/escrow` | ✅ | Used by `src/features/escrow` and `src/providers/EscrowProviders.tsx`. Mounted only where escrow is needed |
| `@apollo/client` | ❌ | Integrated app only |
| `react-icons` | ❌ | Use `lucide-react` |
| `@stellar/freighter-api` | ❌ | Integrated app only |

> **Basis:** `package.json`, `src/features/escrow`, and `src/providers/EscrowProviders.tsx` at commit `ad2f38d7`.
> **Not yet verified:** `@apollo/client` is not imported by `src/features/apartments`, and `@stellar/freighter-api` is absent from `package.json`.
