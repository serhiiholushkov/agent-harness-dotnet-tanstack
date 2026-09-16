---
name: regenerate-api-client
description: 'Run the OpenAPI → TypeScript codegen pipeline, read the contract diff, curate re-exports, and fix drift-check failures. Use after any API endpoint or Response change, when CI fails the drift check, when web typecheck breaks on api-client types, or when schema names degrade (Response2, unknown). Trigger terms: regenerate client, openapi diff, drift check, api-client, generated types, codegen.'
---

# Regenerate the API client

The pipeline is `C# → dotnet build → apps/api/openapi/*.json → openapi-typescript → packages/api-client/src/*.gen.ts`. Contract changes are made **only** in C#; both outputs are committed ([codegen-pipeline](../../rules/monorepo/codegen-pipeline.md)).

## Procedure

1. Regenerate:

   ```bash
   pnpm turbo run build generate
   ```

   Turbo orders `@repo/api#build` (emits the documents) before `@repo/api-client#generate`. The root alias `pnpm generate` runs the same pipeline — the `generate` task depends on `@repo/api#build`.

2. Review the `openapi/*.json` diff — **it is the contract change** and the PR's primary review artifact. Check:
   - only the operations you meant to change appear;
   - new operations carry the intended operation id, tag, and every declared status;
   - schema names are slice-prefixed (`GetProductResponse`), required/nullable match the C# annotations;
   - no accidental breaking change to an active version (removed field, type change, tightened requirement);
   - health routes and the reference UI do not appear.
3. Review the `*.gen.ts` diff for the same story told in TypeScript. Never edit these files.
4. Curate: if the web needs new DTOs, add stable re-exports in `packages/api-client/src/index.ts` — `export type ProductV1 = components['schemas']['GetProductResponse']`. Features never index into `components[…]` directly.
5. Confirm cleanliness — exactly what CI runs:

   ```bash
   pnpm turbo run build generate
   git diff --exit-code -- apps/api/openapi packages/api-client/src
   ```

6. Run `pnpm typecheck`: web compile errors after regeneration are the pipeline working — fix the adapters/components to the new contract, not the generated files.
7. Commit generated documents + client together with the C# change.

## Symptom → cause → fix

| Symptom                                    | Cause                                                                                       | Fix                                                                                          |
| ------------------------------------------ | ------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| CI drift check fails, local diff non-empty | Endpoint changed without regenerating, or a generated file was hand-edited                  | Revert hand edits; run step 1; commit outputs with the change                                |
| Schemas named `Response`, `Response2`, …   | `CreateSchemaReferenceId` not configured / bypassed                                         | Fix the slice-prefix configuration in `Common.Presentation`; regenerate                      |
| Generated type is `unknown`                | Endpoint returns `Results.Ok(...)` without `Produces<T>` / untyped result                   | Use `TypedResults` + declare `Produces<Response>`; regenerate                                |
| Operation appears in every document        | Endpoint mapped outside a versioned route group (no group name)                             | Map it through `Map{Module}Endpoints`' group                                                 |
| Operation missing from its document        | `IEndpoint.Version` doesn't match the group, or group name typo                             | Align `Version` with the intended `ApiVersions.*`                                            |
| Duplicate operation id error in boot test  | Two endpoints share `WithName`                                                              | Make ids unique and version-suffixed                                                         |
| Enum generated as number union             | `JsonStringEnumConverter` not registered globally                                           | Register it once in `Common.Presentation`; regenerate; treat as versioned change if released |
| Build emits no documents                   | `Microsoft.Extensions.ApiDescription.Server` not wired or `OpenApiDocumentsDirectory` wrong | Point it at `apps/api/openapi`; check `AddOpenApi(name)` per version                         |
| Web typecheck breaks on client types       | The contract legitimately changed                                                           | Update adapters/components; do not patch `*.gen.ts` or hand-write DTOs                       |

## Done when

- [ ] `pnpm turbo run build generate` succeeded; the committed documents and client match the C# exactly (drift check clean).
- [ ] The `openapi/*.json` diff was reviewed and contains only intended changes; no unreviewed breaking change to an active version.
- [ ] New DTOs re-exported under stable names; no feature indexes into generated internals.
- [ ] `pnpm typecheck` passes; generated files contain no hand edits.
