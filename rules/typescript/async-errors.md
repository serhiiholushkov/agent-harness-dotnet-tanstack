---
title: Async and error handling
description: Every outbound call has a timeout and an AbortSignal; failures become typed errors with safe messages.
appliesTo: 'apps/web/**/*.ts, apps/web/**/*.tsx, packages/**/*.ts'
---

# Async and error handling

Outbound calls (server-function adapters and any server-side fetch):

- Every call sets an explicit timeout and passes the request's `AbortSignal` through to the client. A stalled upstream call must not pin an SSR render open until the platform kills it. Combine caller signal + timeout (e.g. `AbortSignal.any([signal, AbortSignal.timeout(ms)])`).
- Retry only idempotent reads (GET), bounded; never retry mutations.
- Handle both channels of `openapi-fetch` results: `error` (documented non-2xx) and thrown failures (network, abort). Both map to one typed error shape.

Typed errors:

- Adapters throw/return a typed `ApiError` carrying `status` and a safe, user-presentable message — never the raw upstream `ProblemDetails` body, headers, or URLs ([../common/errors.md](../common/errors.md)).
- Discriminate error categories the UI treats differently (`not-found`, `unauthorized`, `validation`, `unavailable`) as a union, not by matching message strings.
- Timeout/abort maps to an `unavailable`-class error, not a generic crash.

General:

- No floating promises: every promise is awaited, returned, or explicitly voided with a reason. Async functions that can reject are handled where awaited or delegated to an error boundary.
- No `async` executors inside `new Promise`; no `.then` chains where `await` reads straight-line.
- `Promise.all` for independent work; sequential awaits only when order matters.

Caught by: ESLint (`no-floating-promises` etc.), adapter tests covering timeout and non-2xx mapping, review.
