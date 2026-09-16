---
title: Codegen pipeline
description: One cross-language artifact — committed OpenAPI docs and the client generated from them; never hand-edited.
appliesTo: 'apps/api/openapi/**, packages/api-client/**, turbo.json'
---

# Codegen pipeline

The two languages meet at exactly one place:

```text
C# endpoint metadata + Response records
  → dotnet build (Microsoft.Extensions.ApiDescription.Server)
  → apps/api/openapi/v*.json          (committed)
  → openapi-typescript
  → packages/api-client/src/v*.gen.ts (committed)
  → web server-function adapters
```

Rules:

- C# owns the contract. Contract changes are made in endpoint metadata and `Response` records, then regenerated — never by editing JSON or TypeScript outputs.
- Generated files are **never hand-edited**: `apps/api/openapi/*.json`, `packages/api-client/src/*.gen.ts`, `routeTree.gen.ts`, applied EF Core migrations. Lint rules and CODEOWNERS treat them as read-only.
- Both the documents and the generated client are committed; the `openapi/*.json` diff in a PR **is** the contract change and is reviewed as such.
- The drift check is part of `verify` and CI:

  ```bash
  pnpm turbo run build generate
  git diff --exit-code -- apps/api/openapi packages/api-client/src
  ```

  A non-empty diff means an endpoint changed without regeneration, or a generated file was hand-edited. Both fail the build.

- `packages/api-client` stays application-neutral: `client.ts` is a config-free `openapi-fetch` factory taking base URL, timeout, credential and correlation hooks as arguments; the package reads no configuration.
- `index.ts` re-exports the DTOs the web uses under stable versioned names (`ProductV1`); features never index into `*.gen.ts` internals.
- One generated module per API version (`v1.gen.ts`, `v2.gen.ts`), one OpenAPI document per version.
- Generated files need no tests.

Caught by: the drift check, ESLint read-only overrides on `*.gen.ts`, CODEOWNERS, review of the committed document diff.
