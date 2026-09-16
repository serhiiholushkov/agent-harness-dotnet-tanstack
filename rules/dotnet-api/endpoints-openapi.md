---
title: Endpoints and OpenAPI metadata
description: Every public endpoint declares complete, versioned OpenAPI metadata; the generator only sees what is declared.
appliesTo: 'apps/api/src/**/*.cs'
---

# Endpoints and OpenAPI metadata

Every public endpoint must set:

- `WithName` — unique, version-suffixed operation id (`getCatalogProductV1`); it becomes the generated client's operation name and must be unique across **all** documents.
- Group name — applied by the module's versioned route group (`WithGroupName(ApiVersions.V1)`), never per endpoint. An endpoint without a group name lands in _every_ document.
- `WithTags` — the module tag from the shared `Tags` class; the only grouping the generator sees.
- `WithSummary`, plus `WithDescription` when the summary is not enough.
- `Produces<T>` / `ProducesProblem` for **every** success and error status the endpoint can return — including 401/403 on protected routes.
- `RequireAuthorization` with a real policy when protected; deprecation metadata when superseded. An endpoint without `RequireAuthorization` is an explicit, reviewed decision — deny by default, never an omission.

Mechanics:

- Use `TypedResults` with declared response types. `Results.Ok(...)` without a declared type generates an untyped schema and `unknown` in TypeScript.
- Routes are relative (`products/{id:guid}`); the group owns `api/v{n}/{module}`.
- `IEndpoint.Version` matches the group the endpoint belongs to; one endpoint maps into exactly one version group.
- Schema reference ids are configured once in `Common.Presentation` to prefix nested types with their declaring slice (`GetProduct.Response` → `GetProductResponse`). Never rename slice `Response` records to dodge collisions.
- Type-mapping decisions are global in `Common.Presentation` and never overridden per endpoint: `JsonStringEnumConverter`, camelCase policy, one null-vs-absent ignore condition, money as integer minor units, `long` beyond 2^53 as string. Full table: [../../docs/architecture.md](../../docs/architecture.md#type-mapping-across-the-boundary).
- Health routes (`/health/live`, `/health/ready`) call `ExcludeFromDescription()`; the Scalar UI at `/docs` is disabled or protected outside non-production.
- Versioning: URL major versions only; breaking changes add a v2 endpoint beside v1 ([../../docs/architecture.md](../../docs/architecture.md#endpoint-versioning)).

Caught by: the OpenAPI boot test (documents generate, operation ids unique, routes present exactly once), the committed `openapi/*.json` diff in review, CI drift check.
