---
name: add-web-feature
description: 'Add or extend a web feature: feature folder, TanStack server-function adapters over the generated client, queryOptions factories and mutation hooks, components, and a thin route. Use when building UI for an API capability, adding a page/screen, wiring TanStack Query, or creating server functions. Trigger terms: web feature, page, screen, route, server function, useSuspenseQuery, mutation, TanStack.'
---

# Add a web feature

Builds the web side of a capability on top of regenerated client types — run [regenerate-api-client](../regenerate-api-client/SKILL.md) first if the API changed. Standards: [features](../../rules/web/features.md), [server-functions](../../rules/web/server-functions.md), [tanstack-query](../../rules/web/tanstack-query.md), [routes](../../rules/web/routes.md).

## Decision points

- **New feature folder or existing?** Name folders after product concepts; a screen composing two API modules is still one feature. Extending an existing concept extends its folder.
- **Read, write, or both?** Reads need adapter + `queryOptions` + suspense component; writes add a mutation hook + invalidation keys.
- **Which DTOs?** Only curated re-exports from `@repo/api-client` (`ProductV1`). Missing one → add it to the client's `index.ts`, not to `types.ts`.
- **Shared UI?** Generic primitives go to `packages/ui`; product-aware components stay in the feature.

## Procedure

Full worked example (adapter, queries, component, route, tests): [references/feature-example.md](references/feature-example.md).

1. **Scaffold** `apps/web/src/features/<feature>/` with `components/`, `queries/`, `server/`, `index.ts` (+ `hooks/`, `types.ts` when needed). The barrel exports only what routes/siblings consume.
2. **Server-function adapters** in `server/` — one per operation, kebab-case file per function. Each: `createServerFn` + `.validator(...)`, calls the generated client from `@/lib/api-client`, passes `signal` + explicit timeout, forwards correlation headers, maps failures to typed errors. No business logic, no config reads.
3. **Query layer** in `queries/` — one `queryOptions` factory object per resource with structured keys (`['catalog', 'products', id]`, normalized filters); query functions delegate to adapters. Mutation hooks wrap `useServerFn` and invalidate the narrowest keys on success.
4. **Components** in `components/` — read with `useSuspenseQuery(factory)`, render from typed data, use `packages/ui` primitives, meet the a11y floor ([components-styling](../../rules/web/components-styling.md)).
5. **Route** — validate search params, `loader: ({ context }) => context.queryClient.ensureQueryData(factory(...))`, render the feature entry from the barrel. Nothing else in the route file.
6. **Tests**, colocated ([web-testing](../../rules/testing/web-testing.md)):
   - adapter: success, timeout, non-2xx → typed error, mocked at the HTTP boundary with generated types;
   - queries: key structure, adapter delegation, mutation invalidation;
   - components: rendered states and interactions by role/label.
7. Run `pnpm verify`.

## Example (compact)

```ts
// features/catalog/queries/products.queries.ts
export const productQueries = {
  detail: (id: string) =>
    queryOptions({
      queryKey: ['catalog', 'products', id],
      queryFn: ({ signal }) => getProduct({ data: id, signal }),
    }),
};

// routes/catalog/$productId.tsx
export const Route = createFileRoute('/catalog/$productId')({
  loader: ({ context, params }) =>
    context.queryClient.ensureQueryData(productQueries.detail(params.productId)),
  component: () => {
    const { productId } = Route.useParams();
    return <ProductDetail productId={productId} />;
  },
});

// features/catalog/components/product-detail.tsx
export function ProductDetail({ productId }: { productId: string }) {
  const { data: product } = useSuspenseQuery(productQueries.detail(productId));
  return <h1>{product.name}</h1>;
}
```

## Done when

- [ ] Feature folder owns adapters, queries, components; externals import only the barrel; route stays thin.
- [ ] Adapters: generated client + timeout + signal + correlation headers + typed errors; browser never sees the API base URL.
- [ ] Loader and components share the same `queryOptions` factories; mutations invalidate the narrowest keys.
- [ ] DTOs come from `@repo/api-client` re-exports; `types.ts` holds UI-only types.
- [ ] Colocated tests cover adapter error mapping (incl. timeout), query keys/invalidation, and component behavior.
- [ ] `pnpm verify` passes.
