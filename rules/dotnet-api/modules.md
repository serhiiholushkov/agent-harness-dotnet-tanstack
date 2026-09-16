---
title: Modules
description: Two-project module shape, internal by default, explicit registration, minimal Contracts.
appliesTo: 'apps/api/src/**/*.cs'
---

# Modules

Shape:

- A module is exactly two projects: `{Module}` (implementation, everything `internal`) and `{Module}.Contracts` (its only public surface). No Domain/Application/Infrastructure layer projects inside a module.
- The implementation contains `Features/`, `Database/`, `PublicApi/`, and `{Module}Module.cs`. Its only public members are `Add{Module}Module(IServiceCollection, IConfiguration)` and `Map{Module}Endpoints(IEndpointRouteBuilder)`.
- Adding a module is ADR-gated ([../../docs/architecture.md](../../docs/architecture.md#adr-triggers)); follow the add-module skill, Contracts first.

Registration:

- Modules are listed explicitly in `Program.cs`, in dependency order — never discovered by a host-level scan.
- Each module scans **its own** assembly for handlers/validators/endpoints with `publicOnly: false` and `includeInternalTypes: true`, and calls `services.Decorate` over its own registrations. A host-level scan or decorate silently breaks module boundaries or undecorates the last-registered module.
- `Map{Module}Endpoints` owns the route prefix and one route group per active version, applying prefix + module tag + OpenAPI group name; `MapEndpoints(version)` maps only matching `IEndpoint.Version` implementations.

Contracts:

- Contracts hold only: the `I{Module}Api` interface, purpose-built DTO records, and integration event types. No entity, no `DbSet`, no EF type, no error catalogue, no logic.
- `{Module}.Contracts` references nothing except (at most) `Common.SharedKernel`.
- Keep contracts small: a method per consumer question is a query API onto someone else's database — prefer an integration event and a local copy.
- Integration events are versioned public data: add properties, never rename or repurpose them once a consumer exists.
- The `PublicApi/` implementation goes through the module's own slice handlers — never straight to the `DbContext` — and returns `null`/plain DTOs, never the module's `Result` errors.

Caught by: compiler (`internal` across assemblies), NetArchTest module-isolation/Contracts-purity/host-purity tests, review.
