---
title: Web features
description: Feature folders own their UI, queries, adapters, and types; the barrel is the only public surface.
appliesTo: 'apps/web/src/**/*.ts, apps/web/src/**/*.tsx'
---

# Web features

Ownership — each `apps/web/src/features/<feature>/` owns:

| Folder / file | Holds                                                                                        |
| ------------- | -------------------------------------------------------------------------------------------- |
| `components/` | The feature's UI, with colocated `.test.tsx`                                                 |
| `queries/`    | `queryOptions` factories and mutation hooks (`*.queries.ts`)                                 |
| `hooks/`      | Feature-specific hooks                                                                       |
| `server/`     | TanStack server functions — HTTP adapters only ([server-functions.md](server-functions.md))  |
| `types.ts`    | UI-only state and types — never wire DTOs ([../typescript/types.md](../typescript/types.md)) |
| `index.ts`    | The feature's public API; the only entry point for outsiders                                 |

Rules:

- External consumers import only `@/features/<feature>`. Feature internals use relative imports; deep imports across features are forbidden.
- Sibling features are composed in routes or app-level `components/`, not by reaching into each other. A justified direct sibling dependency uses only the sibling's barrel.
- Features are named after product concepts, not API modules; a screen may compose data from two modules under one feature.
- App-wide, feature-neutral code lives in `lib/` (e.g. the configured API client) and `components/` (cross-feature composition). Generic UI primitives live in `packages/ui`; anything with product meaning stays in the feature.
- Nothing in a feature reads configuration directly; server-only config comes through the validated config module in `lib/`.
- Business logic lives in feature hooks/components acting on typed data — never in route files, never in server-function adapters.

Caught by: ESLint boundaries (feature barrels, `packages/*` edges), review.
