---
name: version-endpoint
description: 'Handle a breaking change to a released API endpoint: decide whether it is actually breaking, add a v2 endpoint or slice beside v1, register the new OpenAPI document and route group, deprecate and eventually retire the old version. Use when changing a released contract — removing/renaming fields, changing types or status semantics, changing paths — or when asked to add /api/v2. Trigger terms: breaking change, versioning, v2, deprecate endpoint, sunset, retire API.'
---

# Version an endpoint

Released contracts are immutable; breaking changes add a version beside the old one. Standards: [endpoints-openapi](../../rules/dotnet-api/endpoints-openapi.md), [architecture: versioning](../../docs/architecture.md#endpoint-versioning).

## Decision tree

1. **Is it breaking?** Breaking: removing/renaming a field, changing its meaning or type, making optional input required, changing status semantics, changing the path, adding an enum member a strict client switches on. Not breaking: adding an optional response field a generated consumer can ignore, adding a new endpoint.
   - Not breaking → change in place within the current version; regenerate; done (no new version).
2. **Only the wire shape changed** (same use case, new contract) → add a **second `Endpoint` nested type in the same slice**, mapped into the v2 group, reusing the same handler.
3. **The use case itself changed** → add a **new slice file** beside the old one. The two slices share the entity's rules, never each other's handlers.
4. Never: mutate the released document, clone the module for v2, or mix URL/header/query versioning. Major versions live in the URL only.

## Procedure

1. **First v2 in the whole API?** Add `ApiVersions.V2` in `Common.Presentation`, register the document `AddOpenApi("v2")` in the host, and confirm the api-client `generate` script and Turbo `outputs` glob (`src/*.gen.ts`, `openapi/*.json`) cover the new files.
2. **First v2 in this module?** Add a v2 route group in `Map{Module}Endpoints`:

   ```csharp
   RouteGroupBuilder v2 = app.MapGroup("api/v2/catalog")
       .WithTags(Tags.Catalog)
       .WithGroupName(ApiVersions.V2);

   v2.MapEndpoints(ApiVersions.V2);
   ```

3. Add the new endpoint (per the tree above) with `Version => ApiVersions.V2`, a v2 operation id (`getCatalogProductV2`), full metadata, and its own `Response` shape. Example of the second-endpoint variant:

   ```csharp
   // inside GetProduct — v1 Endpoint stays untouched
   internal sealed record ResponseV2(Guid Id, string Name, MoneyV2 Price); // new wire shape

   internal sealed class EndpointV2 : IEndpoint
   {
       public string Version => ApiVersions.V2;

       public void MapEndpoint(IEndpointRouteBuilder app) =>
           app.MapGet("products/{id:guid}", /* map handler result to ResponseV2 */)
           .WithName("getCatalogProductV2")
           .WithSummary("Get a catalog product")
           .Produces<ResponseV2>(StatusCodes.Status200OK)
           .ProducesProblem(StatusCodes.Status404NotFound)
           .RequireAuthorization();
   }
   ```

4. Keep v1 mapped, tested, and **mark it deprecated** in its endpoint metadata. When retirement is scheduled, also return `Deprecation`, `Sunset`, and successor `Link` headers from the v1 endpoint.
5. Regenerate via [regenerate-api-client](../regenerate-api-client/SKILL.md) — expect a new `v2.json` + `v2.gen.ts` and a deprecation-only diff on v1. Add curated re-exports (`ProductV2`) in the client's `index.ts`.
6. Migrate web adapters operation by operation to the v2 client module; v1 and v2 coexist while clients move.
7. **Retire** only when telemetry confirms no supported client calls v1 and the deprecation window ended (ADR-recorded schedule). Delete the v1 endpoint (and slice, if the use case moved), regenerate — the web-side compile error is the expected way to find stragglers — and remove the empty group/document when the whole version is gone.

## Done when

- [ ] Breaking-change classification stated in the PR; non-breaking changes did not spawn a version.
- [ ] v2 endpoint/slice added beside v1; handlers not shared across use cases; v1 untouched except deprecation metadata.
- [ ] Both documents generate; operation ids unique across documents; v1 shows only deprecation in its diff.
- [ ] v1 integration tests still pass; v2 has its own tests.
- [ ] Client regenerated; re-exports added; web adapters migrated (or migration plan recorded).
- [ ] Retirement schedule ADR-recorded; `pnpm verify` passes.
