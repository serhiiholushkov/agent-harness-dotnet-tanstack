# Web feature example

Full reference implementation for a `catalog` feature slice of UI. Substitute concepts; keep shapes.

## Shared client instance (once per app, in lib/)

```ts
// apps/web/src/lib/api-client.ts
import { createApiClient } from '@repo/api-client';
import { getRequestHeader } from '@tanstack/react-start/server';
import { serverConfig } from './config'; // validated, server-only

export const apiClient = createApiClient({
  baseUrl: serverConfig.apiBaseUrl,
  timeoutMs: serverConfig.apiTimeoutMs,
  headers: () => ({
    authorization: `Bearer ${serverConfig.apiToken}`,
    ...forwardCorrelation(),
  }),
});

function forwardCorrelation() {
  const traceparent = getRequestHeader('traceparent');
  const requestId = getRequestHeader('x-request-id');
  return {
    ...(traceparent ? { traceparent } : {}),
    ...(requestId ? { 'x-request-id': requestId } : {}),
  };
}
```

The factory lives in `@repo/api-client` and reads no configuration itself; the web app owns the values. This file is server-only — importing it from a component is an error.

## Server-function adapters

```ts
// apps/web/src/features/catalog/server/get-product.ts
import { createServerFn } from '@tanstack/react-start';
import { apiClient } from '@/lib/api-client';
import { toApiError } from '@/lib/api-error';

export const getProduct = createServerFn({ method: 'GET' })
  .validator((id: string) => id)
  .handler(async ({ data: id, signal }) => {
    const { data, error, response } = await apiClient.GET(
      '/api/v1/catalog/products/{id}',
      { params: { path: { id } }, signal },
    );

    if (error) {
      throw toApiError(response.status, error);
    }

    return data;
  });
```

```ts
// apps/web/src/features/catalog/server/create-product.ts
import { createServerFn } from '@tanstack/react-start';
import { apiClient } from '@/lib/api-client';
import { toApiError } from '@/lib/api-error';

type CreateProductInput = { name: string; priceCents: number };

export const createProduct = createServerFn({ method: 'POST' })
  .validator((input: CreateProductInput) => input)
  .handler(async ({ data, signal }) => {
    const {
      data: id,
      error,
      response,
    } = await apiClient.POST('/api/v1/catalog/products', {
      body: data,
      signal,
    });

    if (error) {
      throw toApiError(response.status, error);
    }

    return id;
  });
```

`toApiError` maps status + documented error body to the typed `ApiError` union (`not-found`, `validation`, `unauthorized`, `unavailable`) with a safe message — never the raw `ProblemDetails`.

## Query layer

```ts
// apps/web/src/features/catalog/queries/products.queries.ts
import {
  queryOptions,
  useMutation,
  useQueryClient,
} from '@tanstack/react-query';
import { useServerFn } from '@tanstack/react-start';
import { getProduct } from '../server/get-product';
import { createProduct } from '../server/create-product';

export const productQueries = {
  all: () => ['catalog', 'products'] as const,
  detail: (id: string) =>
    queryOptions({
      queryKey: [...productQueries.all(), id],
      queryFn: ({ signal }) => getProduct({ data: id, signal }),
    }),
};

export function useCreateProduct() {
  const queryClient = useQueryClient();
  const createFn = useServerFn(createProduct);

  return useMutation({
    mutationFn: createFn,
    onSuccess: () =>
      queryClient.invalidateQueries({ queryKey: productQueries.all() }),
  });
}
```

## Component

```tsx
// apps/web/src/features/catalog/components/product-detail.tsx
import { useSuspenseQuery } from '@tanstack/react-query';
import { productQueries } from '../queries/products.queries';
import { formatPrice } from '../lib/format-price';

export function ProductDetail({ productId }: { productId: string }) {
  const { data: product } = useSuspenseQuery(productQueries.detail(productId));

  return (
    <article aria-labelledby="product-name">
      <h1 id="product-name">{product.name}</h1>
      <p>{formatPrice(product.priceCents)}</p>
    </article>
  );
}
```

## Barrel and route

```ts
// apps/web/src/features/catalog/index.ts
export { ProductDetail } from './components/product-detail';
export { productQueries, useCreateProduct } from './queries/products.queries';
```

```tsx
// apps/web/src/routes/catalog/$productId.tsx
import { createFileRoute } from '@tanstack/react-router';
import { ProductDetail, productQueries } from '@/features/catalog';

export const Route = createFileRoute('/catalog/$productId')({
  loader: ({ context, params }) =>
    context.queryClient.ensureQueryData(
      productQueries.detail(params.productId),
    ),
  component: RouteComponent,
});

function RouteComponent() {
  const { productId } = Route.useParams();
  return <ProductDetail productId={productId} />;
}
```

## Tests

```ts
// apps/web/src/features/catalog/server/get-product.test.ts
// Mock at the HTTP boundary; type fixtures from the generated client.
import type { ProductV1 } from '@repo/api-client';

const product: ProductV1 = {
  id: 'a1…',
  name: 'Widget',
  priceCents: 1999,
  isArchived: false,
};
// assert: 200 → returns data; 404 → throws ApiError{kind:'not-found'};
// timeout/abort → throws ApiError{kind:'unavailable'}; body never contains upstream ProblemDetails
```

```tsx
// apps/web/src/features/catalog/components/product-detail.test.tsx
// Render with a test QueryClient (retries off), seed productQueries.detail(id) with a typed fixture,
// assert by role/heading text — not test ids.
```

A contract change regenerates `ProductV1`, and these fixtures fail to compile — that is the point.
