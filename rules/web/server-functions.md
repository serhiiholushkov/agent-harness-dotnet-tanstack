---
title: Server functions
description: TanStack server functions are the only path to the API — thin, typed, correlated, server-only.
appliesTo: 'apps/web/src/**/server/**/*.ts, apps/web/src/lib/**/*.ts'
---

# Server functions

Server functions under `features/<feature>/server/` are backend-for-frontend **adapters only**:

- They are the single lawful path from web code to the API. Components, hooks, loaders, and routes never fetch the API directly; the browser calls only the server-function RPC stub.
- Each adapter calls the API through the generated `openapi-fetch` client instantiated in `lib/` — so a renamed path or changed payload is a TypeScript error, not a runtime 404.
- The API base URL and service credential come from the validated, server-only config module in `lib/`; they never appear in client-bundled code, and no `import.meta.env` reads happen inside features ([../common/security.md](../common/security.md)).
- Every call sets an explicit timeout and forwards the request's `AbortSignal`; retry only idempotent reads ([../typescript/async-errors.md](../typescript/async-errors.md)).
- Forward the incoming `traceparent` and `x-request-id` headers so one correlation id spans web and API.
- Attach credentials server-side (session exchange/forwarding); the browser never holds an API token.
- Failures map to typed errors with a status and safe message; the upstream `ProblemDetails` body never reaches the browser.
- Validate input with `.validator(...)` so the RPC boundary rejects malformed data before it travels.

Forbidden inside a server function:

- Business logic, data shaping beyond transport mapping, or composition of multiple use cases — that pressure means the API needs a better endpoint.
- Database access of any kind.
- Reading raw env/config, logging secrets, or echoing upstream error bodies.

Caught by: adapter tests (success, timeout, non-2xx mapping) mocked at the HTTP boundary with generated types, ESLint boundaries, security review.
