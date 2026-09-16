---
title: C# language discipline
description: net10.0 project settings, records vs classes, primary constructors, visibility defaults.
appliesTo: 'apps/api/**/*.cs'
---

# C# language discipline

Project settings (fixed in `Directory.Build.props`, never overridden per project):

- `net10.0`, `ImplicitUsings` and `Nullable` enabled.
- `TreatWarningsAsErrors=true`, `AnalysisMode=All`, `AnalysisLevel=latest`, `EnforceCodeStyleInBuild=true`. Do not suppress a warning to pass the build; fix it or justify the suppression inline with a reason.
- Central package management: every version lives in `Directory.Packages.props`; a `.csproj` never carries a `Version` attribute.

Types:

- `sealed` by default for classes; `internal` by default inside module implementation projects — `public` is reserved for Contracts, `Common.*` surfaces, and module entry points.
- Records for messages and DTOs: `Command`, `Query`, `Response`, Contracts DTOs (`ProductSummary`) are `sealed record`s — immutable, value-equal.
- Classes for entities (private setters, behavior methods) and for handlers/validators/endpoints.
- Primary constructors for dependency injection: `internal sealed class Handler(CatalogDbContext context) : IQueryHandler<…>`.
- One type per file; file-scoped namespaces; the file named after the type ([../common/naming.md](../common/naming.md)).

Style points the reference code assumes:

- Explicit types where the samples use them (`Result<Response> result = await …`); pattern matching (`is null`, `is { } x`, switch expressions) over `== null` chains.
- No static mutable state; time comes from `IDateTimeProvider`, never `DateTime.UtcNow` in a handler.
- No reflection tricks or `InternalsVisibleTo` except a module's own test projects.

Caught by: build (warnings as errors), analyzers, `dotnet format Api.slnx --verify-no-changes`, architecture convention tests (sealed/internal).
