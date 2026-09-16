---
title: Result usage
description: Expected failures are Result values with typed errors; exceptions are for bugs.
appliesTo: 'apps/api/**/*.cs'
---

# Result usage

- Handlers return `Result` (commands) or `Result<T>` (queries and value-returning commands). Every expected failure is `Result.Failure(SomeErrors.X(…))`; exceptions are reserved for genuine bugs and infrastructure faults the global handler catches.
- `Error` carries a stable code, a message, and an `ErrorType`. `ErrorType` members map one-to-one to promised HTTP statuses — including `Unauthorized` and `Forbidden`; see [../common/errors.md](../common/errors.md) for the mapping.
- Error catalogues are static classes per feature entity (`ProductErrors.NotFound(id)`), `internal` to the module. Never construct ad hoc `Error` instances inline in a handler for a reusable condition, and never reference another module's catalogue.
- Endpoints consume results exactly once, via `result.Match(TypedResults.Ok, CustomResults.Problem)` (or the appropriate success factory). No `if (result.IsFailure)` HTTP branching in endpoints.
- Domain methods on entities that can fail return `Result` too (`product.Archive()`); handlers propagate rather than translate them.
- Do not wrap `Result` in exceptions or exceptions in `Result` for control flow. A caught infrastructure exception either bubbles (bug) or is mapped at a boundary adapter with a deliberate `Error`.
- Every `Result.Failure` branch in a handler has a unit test ([../testing/api-testing.md](../testing/api-testing.md)).

Caught by: handler unit tests per failure branch, integration tests on status contracts, review.
