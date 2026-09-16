---
title: Security
description: Secret handling, the trust boundary between web and API, and the authorization floor.
appliesTo: '**/*'
---

# Security

Secrets:

- Secrets arrive as environment variables or from a secret store; local development uses .NET user secrets / untracked `.env`. Never committed, logged, echoed in CI, or returned by an endpoint.
- The API base URL and service credential are server-only web configuration; they never reach a client bundle, and the browser never holds an API bearer token.
- Redact `Authorization`, `Cookie`, and any password or secret field before logging. EF Core sensitive-data logging stays off outside local development.

Trust boundary:

- The browser holds an httpOnly, `SameSite=Lax`-or-stricter session cookie issued by the web app; server functions exchange or forward it for the API credential on every call.
- Server functions are POST RPC endpoints: verify request origin; a router `beforeLoad` guard is UX, not authorization.
- The API validates the token itself on every protected endpoint — it never trusts the web tier.

Authorization floor (API):

- Every protected endpoint calls `RequireAuthorization` with a real policy. A permission that resolves to "any authenticated principal" is scaffolding, not authorization.
- Every handler enforces resource ownership itself from `IUserContext`, as part of the query filter. The owner is never accepted from the request.
- Deny by default; `401`/`403` payloads are consistent and do not reveal whether the resource exists.

Hardening:

- Kestrel limits are explicit: max request body size, request headers timeout, keep-alive timeout.
- `UseForwardedHeaders` only behind a proxy that rewrites `X-Forwarded-For`, with known proxies configured.
- The built-in rate limiter is per-process; multi-instance limits live at the gateway or a distributed store (recorded choice).
- CSP, frame options, and referrer policy belong to the web app; the API sends `nosniff` and HSTS from `Common.Presentation`.
- CORS is registered only if a browser calls the API directly — the default server-function path needs none.

Caught by: integration tests on 401/403 contracts, ownership tests per slice, security reviewer, secret scanners in CI, review.
