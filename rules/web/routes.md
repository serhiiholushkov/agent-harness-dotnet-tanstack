---
title: Routes
description: Route files wire URLs to features: validate, prefetch, render. Nothing else.
appliesTo: "apps/web/src/routes/**/*.ts, apps/web/src/routes/**/*.tsx"
---

# Routes

A file under `apps/web/src/routes/**` does exactly three things:

1. **Validate URL state** — parse and validate search params and path params into typed values (`validateSearch`), so features receive typed input, never raw strings.
2. **Prefetch** — the loader calls `context.queryClient.ensureQueryData(someQueryOptions(...))` using the same `queryOptions` factories components read with; SSR prefetch and hydration must share one key and function.
3. **Render** — compose feature entry points imported from feature barrels (`@/features/<feature>`).

Forbidden in route files:

- Fetch logic, server-function definitions, or direct `@repo/api-client` usage — those live in the feature's `server/` and `queries/`.
- Business logic, data transformation, or UI beyond composition.
- Ad hoc query keys or inline query functions.
- Hand edits to `routeTree.gen.ts` — it is generated.

Guards:

- `beforeLoad` may redirect unauthenticated users as a UX affordance; it is **not** an authorization boundary. The API authorizes every request itself; mutations are protected by cookie policy + origin checks, not by UI guards ([../common/security.md](../common/security.md)).

Error and pending UI:

- Routes declare error and pending components rendering from typed errors ([../typescript/async-errors.md](../typescript/async-errors.md)); raw transport objects never render.

Caught by: ESLint boundaries (route files restricted to feature barrels), review.
