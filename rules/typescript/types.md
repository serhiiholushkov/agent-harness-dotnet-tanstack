---
title: Type sourcing and modeling
description: DTOs come only from the generated client; UI state is modeled with discriminated unions.
appliesTo: 'apps/web/**/*.ts, apps/web/**/*.tsx'
---

# Type sourcing and modeling

DTO sourcing:

- Every type that crosses the HTTP boundary comes from `@repo/api-client` — the curated re-exports in its `index.ts` (`ProductV1`), never `components['schemas'][…]` indexing inside features.
- Copying or re-declaring a generated DTO shape by hand — in `types.ts`, a component prop, or a test — is a boundary violation. When the API contract changes, regenerate ([../monorepo/codegen-pipeline.md](../monorepo/codegen-pipeline.md)); do not patch types locally.
- A feature's `types.ts` holds UI-only state: view models, filter/sort descriptors, component prop unions. If a type mirrors a wire schema, it belongs to the generated client.
- Derive variations instead of redefining: `Pick<ProductV1, 'id' | 'name'>`, mapped types over the DTO — so contract changes propagate as compile errors.

Modeling:

- Model state machines and adapter outcomes as discriminated unions with a literal tag, and switch exhaustively (`never` check in the default arm):

  ```ts
  type ProductsState =
    | { status: 'loading' }
    | { status: 'error'; error: ApiError }
    | { status: 'ready'; products: ProductV1[] };
  ```

- Generated API enums arrive as string unions (the API registers `JsonStringEnumConverter`); switch on them exhaustively so a new member is a compile error, and treat new members as a versioned API change.
- No boolean flag pairs (`isLoading` + `isError`) where a union states the truth.

Caught by: `pnpm typecheck` (exhaustiveness), ESLint boundaries (no deep imports into `*.gen.ts`), review, the drift check.
