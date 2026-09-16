---
title: Boundaries
description: Dependency directions and forbidden edges across the monorepo, and what enforces them.
appliesTo: '**/*'
---

# Boundaries

Allowed dependency chain (full detail: [../../docs/architecture.md](../../docs/architecture.md#dependency-direction)):

```text
apps/web/routes -> web feature public APIs -> feature components/queries/server adapters
web server adapters -> packages/api-client -> HTTP -> apps/api endpoints
apps/api endpoint -> slice handler -> module DbContext -> module PostgreSQL schema
module slice -> another module's Contracts assembly -> that module's public API implementation
apps/web -> packages/ui
packages/api-client -> apps/api/openapi/*.json (build artifact only)
```

Forbidden, with no exceptions outside an ADR:

- `packages/*` referencing `apps/*` source.
- `apps/web` referencing C# source, connection strings, or the database.
- An endpoint referencing a `DbContext` — the handler owns data access.
- A slice referencing another slice's `Command`, `Query`, `Handler`, `Validator`, or `Endpoint`.
- A module referencing another module's implementation assembly, entities, `DbContext`, schema, or error catalogue — Contracts and integration events are the only doors.
- `{Module}.Contracts` referencing anything except `Common.SharedKernel`.
- The `Api` host referencing any module type other than `Add{Module}Module` and `Map{Module}Endpoints`.
- `Common.*` referencing any module namespace; `packages/*` or `Common.*` holding business logic.
- Module-to-module HTTP loopback, `HttpClient` against the same host, or reusing the generated TypeScript client inside the API.
- Hand-written TypeScript DTOs duplicating a schema `packages/api-client` already generates.

Enforcement:

- Compiler: modules are separate assemblies with `internal` types; cross-module references fail the build.
- Architecture tests (NetArchTest): module isolation both directions, slice isolation, Contracts purity, host purity.
- ESLint boundaries on the web: feature barrels, package edges.
- CI drift check: catches hand-edited or stale generated contract files.

What no tool catches — cross-schema SQL strings and foreign keys in migrations — is a mandatory review item; see [../dotnet-api/data-ownership.md](../dotnet-api/data-ownership.md).
