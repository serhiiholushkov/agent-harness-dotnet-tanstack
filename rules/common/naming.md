---
title: Naming
description: Names for files, types, routes, operation ids, keys, and tests across both stacks.
appliesTo: '**/*'
---

# Naming

C# (API):

- File name equals the type it holds; a slice file is named after its use case verb-noun: `GetProduct.cs`, `CreateProduct.cs`.
- Module projects: `{Module}` and `{Module}.Contracts`; module entry: `{Module}Module.cs` exposing `Add{Module}Module` / `Map{Module}Endpoints`.
- Per-module types: `{Module}DbContext`, `{Entity}Errors`, `{Module}CacheKeys`, options `{Name}Options` with a `SectionName` constant.
- PostgreSQL: schema per module named after the module (`catalog`), snake_case tables/columns via `UseSnakeCaseNamingConvention()`.
- Nested slice types are always `Command`/`Query`, `Response`, `Validator`, `Handler`, `Endpoint` — the slice class carries the meaning.
- Contract methods are `Async`-suffixed (`GetAsync`, `GetManyAsync`).

HTTP:

- Routes: `/api/v{n}/{module}/{plural-resource}` — version and module prefix come from the route group; endpoints declare only the relative part (`products/{id:guid}`).
- Operation ids (`WithName`): camelCase verb-noun with version suffix — `getCatalogProductV1`. Unique across all documents.
- JSON properties: camelCase (ASP.NET Core default policy; never customized per endpoint).

Web:

- Files kebab-case: `product-list.tsx`, `get-product.ts`, `products.queries.ts`.
- Feature folders named after the product concept (`catalog`), not after API modules.
- Feature layout: `components/`, `queries/`, `hooks/`, `server/`, `types.ts`, `index.ts`.
- Workspace packages: `@repo/api-client`, `@repo/ui`, `@repo/eslint-config`, `@repo/typescript-config`, `@repo/api`, `@repo/web`.
- Curated DTO re-exports carry a version: `export type ProductV1 = components['schemas']['GetProductResponse']`.

Tests:

- Web: `<source-name>.test.ts` / `.test.tsx`, colocated. No `__tests__` folders.
- API: test classes named after the slice — `GetProductHandlerTests`, `CreateProductValidatorTests` — in projects mirroring `Features/`.

Caught by: `dotnet format` + analyzers, ESLint naming/boundary rules, review against this file.
