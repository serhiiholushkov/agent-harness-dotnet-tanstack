---
title: Nullability
description: Nullable reference types are a contract feeding the OpenAPI schema; no null-forgiving shortcuts.
appliesTo: 'apps/api/**/*.cs'
---

# Nullability

- Nullable reference types stay enabled everywhere. Annotations are contract: a non-nullable property on a `Response` becomes a required property in the OpenAPI schema and a non-optional field in the generated TypeScript. Getting an annotation wrong corrupts the cross-language contract, not just local safety.
- The null-forgiving operator `!` is forbidden in production code paths. The canonical offender — `configuration.GetConnectionString("…")!` — is replaced by validated typed options ([../dotnet-api/configuration.md](../dotnet-api/configuration.md)).
- Model absence explicitly: `Guid?` for an optional external identifier, `ProductSummary?` for a contract lookup that may miss, `Task<IReadOnlyDictionary<Guid, T>>` where missing keys are simply absent.
- Contract lookups return `null` for "not found" rather than leaking the owning module's `Result`/error vocabulary.
- Choose the global JSON ignore condition (`null` vs absent) once in `Common.Presentation` and never override it per endpoint — the generated client's `T | null` vs optional-key shape depends on it.
- Guard clauses at boundaries (`ArgumentNullException.ThrowIfNull` in `Common.*` public surfaces); inside slices rely on the type system instead of defensive null checks.

Caught by: build (nullable warnings are errors), the OpenAPI diff when an annotation changes a schema, review.
