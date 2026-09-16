---
title: Async and cancellation
description: Every I/O path is async and forwards the CancellationToken end to end.
appliesTo: 'apps/api/**/*.cs'
---

# Async and cancellation

- All I/O is `async`/`await`. `.Result`, `.Wait()`, `.GetAwaiter().GetResult()`, and `async void` are forbidden (event handlers on UI frameworks do not exist here).
- Every handler signature takes a `CancellationToken` and forwards it into **every** asynchronous call: EF Core queries, `SaveChangesAsync`, `HybridCache` operations, contract calls, event dispatch. A dropped token is a bug — verify the chain when touching a handler.
- Endpoints bind the request's `CancellationToken` and pass it to the handler; nothing in a slice creates its own token source except to enforce a deliberate timeout.
- Contracts methods are `Async`-suffixed and take `CancellationToken cancellationToken = default` as the last parameter.
- No fire-and-forget `Task.Run` in request paths. Work that must survive the request goes through the outbox and a background processor ([../dotnet-api/data-ownership.md](../dotnet-api/data-ownership.md)).
- Background services honor their stopping token and participate in graceful shutdown (drain flag, `Shutdown:DrainSeconds`).
- `ConfigureAwait(false)` is unnecessary in ASP.NET Core application code; do not scatter it.

Caught by: analyzers (blocking-call and async rules run at `AnalysisMode=All`), review of token forwarding, integration tests that cancel requests.
