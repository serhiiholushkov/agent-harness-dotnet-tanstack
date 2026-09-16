---
name: add-module
description: 'Create a new business module in the .NET API: Contracts project first, internal implementation project, own PostgreSQL schema and DbContext with its own migrations history, explicit registration, and isolation architecture tests. ADR-gated. Use when a new business capability needs its own module, when splitting or extracting a domain, or when someone asks for a new bounded context. Trigger terms: new module, bounded context, new domain, Contracts project, module registration.'
---

# Add a module

A new module is an **ownership decision, not a routine change** — confirm the ADR exists before writing code ([architecture: ADR triggers](../../docs/architecture.md#adr-triggers)). Two entities with a handful of slices are not two modules. Standards: [modules](../../rules/dotnet-api/modules.md), [data-ownership](../../rules/dotnet-api/data-ownership.md).

## Decision points

- **Is it a module?** It has a distinct business capability, an owner, and a describable public surface. If you cannot write its Contracts before its internals, the boundary is not understood — stop and plan.
- **Dependency order.** Which existing modules must register before it (whose Contracts does it consume)? That determines its position in `Program.cs`.
- **First consumers.** What belongs in Contracts now? Start minimal — an empty Contracts project is valid; a wide one is a liability.

## Procedure

Reference vocabulary: new module `Shipping` alongside `Catalog`/`Orders`. Full project files and code: [references/module-scaffold.md](references/module-scaffold.md).

1. **Contracts first** — create `apps/api/src/Modules/Shipping/Shipping.Contracts/` (`Shipping.Contracts.csproj` referencing at most `Common.SharedKernel`). Add only what consumers need today: `IShippingApi`, purpose-built DTO records, integration event types. No entity, no EF type, no error catalogue.
2. **Implementation project** — `apps/api/src/Modules/Shipping/Shipping/` with folders `Features/`, `Database/`, `PublicApi/`, and `ShippingModule.cs`. References: `Shipping.Contracts`, `Common.*`, and the Contracts of modules it consumes. Everything `internal`.
3. **Database identity** — add the schema constant (`Schemas.Shipping = "shipping"`); create `internal sealed ShippingDbContext` with `HasDefaultSchema(Schemas.Shipping)`, `UseSnakeCaseNamingConvention()`, and a migrations history table in its own schema.
4. **Module entry** — `ShippingModule.cs` with the only two public members: `AddShippingModule` (DbContext + own-assembly Scrutor scan with `publicOnly: false` + own `Decorate` calls + `AddValidatorsFromAssembly(..., includeInternalTypes: true)` + `AddEndpoints(assembly)` + `AddScoped<IShippingApi, ShippingApi>`) and `MapShippingEndpoints` (route group `api/v1/shipping` + `WithTags(Tags.Shipping)` + `WithGroupName(ApiVersions.V1)` + `MapEndpoints(ApiVersions.V1)`).
5. **Shared constants** — add `Tags.Shipping` in `Common.Presentation`. Reuse the existing `ApiVersions`.
6. **Host registration** — in `Program.cs`, add `.AddShippingModule(builder.Configuration)` in dependency order and `app.MapShippingEndpoints()`. Add both projects to `Api.slnx`; the host references only the implementation project.
7. **First slice + PublicApi** — add at least one slice via [add-vertical-slice](../add-vertical-slice/SKILL.md); implement `ShippingApi` in `PublicApi/` going through the module's own slice handlers, returning `null`/DTOs, never `Result`.
8. **Architecture tests** — extend `ArchitectureTests` with isolation both directions (no `Shipping` ↔ sibling implementation dependency), Contracts purity (references nothing), and host purity (host uses only the two entry points). Add the module to any test enumerating all modules.
9. **First migration** — run [add-ef-migration](../add-ef-migration/SKILL.md); confirm the generated migration creates the `shipping` schema and history table and adds **no foreign key crossing out of the schema**. External ids are plain `uuid` columns.
10. **Verify** — integration tests boot with the new module; the OpenAPI boot test sees its routes exactly once in their document; `pnpm verify` passes.

## Done when

- [ ] ADR recorded for the module and its dependency position.
- [ ] Two projects; Contracts references ≤ `Common.SharedKernel`; implementation is fully `internal` except the two entry points.
- [ ] Own schema, `DbContext`, and migrations history table; first migration crosses no schema boundary.
- [ ] Registered and mapped explicitly in `Program.cs` in dependency order; own-assembly scan and `Decorate` in place.
- [ ] Architecture tests assert isolation both directions, Contracts purity, host purity.
- [ ] At least one slice, its tests, and the module's decorator-pipeline coverage exist.
- [ ] `pnpm verify` passes; the OpenAPI diff shows only the new module's operations.
