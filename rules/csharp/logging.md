---
title: Logging
description: Structured logging with correlation across both deployables; secrets never logged.
appliesTo: 'apps/api/**/*.cs'
---

# Logging

- Log through `ILogger<T>` with structured message templates: `logger.LogInformation("Product {ProductId} archived", id)`. String interpolation in log calls is forbidden — it destroys structure and cardinality.
- Uniform request/command logging lives in the per-module `LoggingDecorator`; handlers add only domain-meaningful events, not entry/exit noise.
- Correlation: derive the request correlation id from the inbound `x-request-id` when present, otherwise the W3C trace id; emit it on the response. Web server functions forward `traceparent` and `x-request-id`, so one trace spans both deployables.
- OpenTelemetry tracing is enabled for ASP.NET Core, `HttpClient`, and Npgsql in `Common.Infrastructure`; do not hand-roll spans in slices unless measuring a named business operation.
- Never log secrets, `Authorization`/`Cookie` headers, passwords, tokens, connection strings, or full request bodies. Redaction is configured once in `Common.Infrastructure`.
- Log levels: `Information` for business events, `Warning` for expected failures worth attention, `Error` only with an exception or an invariant violation. The global exception handler logs 5xx with full structured context — slices do not double-log rethrown exceptions.
- Log the error code, not the error message chain, for `Result` failures; the message may contain user data.

Caught by: log-output assertions in integration tests where behavior depends on it, telemetry review, security reviewer.
