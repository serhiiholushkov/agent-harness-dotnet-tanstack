---
title: Errors
description: One error shape end to end; expected failures are values; nothing internal leaks.
appliesTo: '**/*'
---

# Errors

API:

- Handlers signal expected failures with `Result`/`Result<T>` carrying an `Error` with a semantically correct `ErrorType`. Exceptions are for bugs, not outcomes.
- Handlers never build HTTP responses; endpoints translate through the one shared `CustomResults.Problem` function. No hand-rolled `ProblemDetails` in slices.
- The `ErrorType` → status mapping is fixed: `Validation`/`Problem` → 400, `Unauthorized` → 401, `Forbidden` → 403, `NotFound` → 404, `Conflict` → 409, `Failure` → 500. Every status the API promises has an explicit `ErrorType` — authorization failures must not fall through to 500.
- Error catalogues are `internal`, per feature, in the owning module (`ProductErrors`). Another module's error codes are never switched on.
- The global `IExceptionHandler` is the last resort: it logs with structured fields, returns the documented `ProblemDetails` shape, preserves 4xx messages, and replaces every 5xx message with a generic one.
- The rate limiter's rejection, the authentication challenge, and endpoint filters return the same content type and shape as the shared handler.
- Stack traces, SQL, Npgsql error codes, tokens, and internal identifiers never appear in a response.

Web:

- Server-function adapters map API failures to typed errors carrying a status and a safe message. An upstream `ProblemDetails` body is never forwarded to the browser.
- Adapter error mapping — including timeout and non-2xx paths — is tested (see [../testing/web-testing.md](../testing/web-testing.md)).
- Route and component error boundaries render from the typed error, never from raw transport objects.

Caught by: handler unit tests per `Result.Failure` branch, integration tests asserting the `ProblemDetails` contract, adapter tests, review.
