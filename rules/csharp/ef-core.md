---
title: EF Core
description: Query, projection, and save rules for per-module DbContexts.
appliesTo: 'apps/api/**/*.cs'
---

# EF Core

Queries:

- Queries project straight to the slice's `Response` (`Select(p => new Response(…))`); tracked entities never reach HTTP, and a projection never leaks a navigation property.
- Read-only entity loads use `AsNoTracking()` when projection has not already untracked them.
- `IQueryable` never crosses out of a handler; materialize inside the slice.
- Ownership filters are part of the query predicate (`… && p.OwnerId == userContext.UserId`), not post-load checks.
- Batch lookups are one `WHERE Id IN (...)` query — never a query per row.
- Raw SQL is allowed inside a slice but may only touch the module's own schema; cross-schema SQL is forbidden ([../dotnet-api/data-ownership.md](../dotnet-api/data-ownership.md)).

Writes:

- One command handler calls `SaveChangesAsync` exactly once; that call is the consistency boundary. No nested or repeated saves per use case.
- Assign identifiers client-side (`Guid.NewGuid()`) before raising domain events, or the event carries `Guid.Empty`.
- Cache invalidation happens after a successful save, never before.
- Uniqueness, in-schema foreign keys, and concurrency are enforced in the database; add concurrency tokens wherever concurrent edits are plausible.

Model and migrations:

- Entity configuration lives in `IEntityTypeConfiguration<T>` classes under the module's `Database/`; entities stay free of mapping attributes.
- `modelBuilder.HasDefaultSchema(Schemas.{Module})` + `UseSnakeCaseNamingConvention()`; the migrations history table lives in the module's schema.
- Migrations are generated per module with the pinned `dotnet-ef` tool and applied as a reviewed, gated step — never `Database.Migrate()` at startup outside tests.
- Applied migrations and the model snapshot are never hand-edited.
- Sensitive-data logging stays off outside local development.

Caught by: integration tests against Testcontainers PostgreSQL with real migrations, migration review, analyzers.
