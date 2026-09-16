---
title: Web testing
description: Colocated Vitest + Testing Library tests, mocked at the HTTP boundary with generated types.
appliesTo: 'apps/web/**/*.test.ts, apps/web/**/*.test.tsx'
---

# Web testing

Placement and naming:

- Every test sits next to the source it covers: `product-list.tsx` → `product-list.test.tsx`; `products.queries.ts` → `products.queries.test.ts`. Use `.test.tsx` only when the test contains JSX. No `__tests__` folders.
- Stack: Vitest + Testing Library. Query DOM by role/label/text (user-visible semantics), not by test id, except where semantics genuinely cannot reach.

What to cover:

- Feature component behavior as the user sees it — rendering states from the discriminated union, interactions, empty/error states.
- Query layer: key structure and stability (normalized filters), fetch delegation to the adapter, and mutation hooks invalidating the narrowest keys.
- Server-function adapters: success parsing, **timeout**, and **non-2xx** mapping to typed errors — the paths that fail loudest in production.
- Route-level composition only where wiring is nontrivial; routes are thin by rule.

Boundaries:

- Mock at the HTTP boundary using the generated types from `@repo/api-client`, so a contract change breaks tests as loudly as it breaks code. Do not mock feature internals or hand-write DTO shapes in fixtures — derive fixture types from `ProductV1` etc.
- Do not mock TanStack Query itself; run it with a test `QueryClient` (retries off) and assert observable results.
- Generated files (`*.gen.ts`, `routeTree.gen.ts`) need no tests.

Caught by: CI running `pnpm test`; type errors in fixtures when the contract changes.
