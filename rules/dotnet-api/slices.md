---
title: Slices
description: One file per use case with nested internal types; slices never couple to each other.
appliesTo: 'apps/api/src/Modules/**/*.cs'
---

# Slices

Anatomy — one use case is one file at `Features/{Entity}/{UseCase}.cs`, holding one `internal static class` with nested types:

| Nested type         | Role                                                                                   |
| ------------------- | -------------------------------------------------------------------------------------- |
| `Command` / `Query` | The input, a `sealed record`, independent of the wire format                           |
| `Response`          | The output DTO — the transport contract the TypeScript is generated from               |
| `Validator`         | FluentValidation input-shape rules (commands; queries validate in-handler when needed) |
| `Handler`           | The use case; injects the module's `DbContext` and ports                               |
| `Endpoint`          | Relative route, `Version`, OpenAPI metadata, `Result` → HTTP mapping                   |

Rules:

- The whole static class is `internal`; nothing in a slice is another module's business.
- A slice never references another slice's `Command`, `Query`, `Handler`, `Validator`, or `Endpoint`. To reuse behavior: duplicate it, push the rule onto the entity, or publish an event — never call the sibling handler.
- A slice may use its own module's entities, error catalogue, cache keys, and `DbContext`, plus `Common.*` and other modules' Contracts. Nothing else.
- Endpoints stay thin: bind, map wire model → command, invoke the injected handler interface, `Match`. Data access, rules, and ownership live in the handler/entity.
- Ownership is part of the handler's query, derived from `IUserContext`; a client-supplied owner id is an authorization hole.
- Validators check input shape; business rules live in the handler or entity. Promote a rule to the entity as soon as a second slice needs it (`product.Archive()` returning `Result`).
- Domain events are raised on the entity after identifiers are assigned; consumers of integration events live in the module that reacts (`OnProductPriceChanged.cs` beside the reacting feature).
- Past ~150 lines, extract to the entity — not to a service, not to another slice.
- Response records are transport contracts: never an entity, never a projection leaking navigation properties, named `Response` (schema ids disambiguate; see [endpoints-openapi.md](endpoints-openapi.md)).

Caught by: NetArchTest slice-isolation and convention tests (sealed/internal), review, the OpenAPI diff for contract drift.
