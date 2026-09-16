---
title: TanStack Query
description: queryOptions factories, structured keys, suspense reads, narrow invalidation, per-request QueryClient.
appliesTo: 'apps/web/src/**/*.ts, apps/web/src/**/*.tsx'
---

# TanStack Query

Factories and keys:

- Every query is defined once as a `queryOptions` factory in the feature's `queries/`, shared by route loaders and components — SSR prefetch and hydration must use the same key and function:

  ```ts
  // apps/web/src/features/catalog/queries/products.queries.ts
  export const productQueries = {
    detail: (id: string) =>
      queryOptions({
        queryKey: ['catalog', 'products', id],
        queryFn: ({ signal }) => getProduct({ data: id, signal }),
      }),
    list: (filters: ProductFilters) =>
      queryOptions({
        queryKey: ['catalog', 'products', 'list', normalizeFilters(filters)],
        queryFn: ({ signal }) => listProducts({ data: filters, signal }),
      }),
  };
  ```

- Query keys are structured arrays — resource, identifier, normalized filters. No ad hoc string keys in components; normalize filter objects so key equality is stable.
- The query function is always a server-function adapter from `server/` — never a direct client call.

Reading and mutating:

- Components read with `useSuspenseQuery(productQueries.detail(id))`; the route loader has already `ensureQueryData`-ed it ([routes.md](routes.md)).
- Mutations use `useServerFn` inside feature mutation hooks; on success invalidate the **narrowest** affected keys (`['catalog', 'products', id]`), not the whole cache.
- Do not duplicate server state into `useState`/`useEffect`; the query cache is the source of truth.

SSR:

- One `QueryClient` per SSR request, created in the router setup with the TanStack Router SSR Query integration installed. A module-level singleton client leaks data between users.

Caught by: query tests (keys, fetch behavior, invalidation) per [../testing/web-testing.md](../testing/web-testing.md), review.
