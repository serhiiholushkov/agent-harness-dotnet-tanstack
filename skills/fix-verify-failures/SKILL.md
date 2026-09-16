---
name: fix-verify-failures
description: 'Triage and fix a red `pnpm verify` or CI run across both toolchains: dotnet format, ESLint/Prettier, tsc, dotnet build/test, Vitest, and the contract drift check. Use when verify, lint, typecheck, tests, build, or the drift check fail, or CI is red. Trigger terms: verify failed, CI red, drift check, build error, test failure, lint error, typecheck error.'
---

# Fix verify failures

`pnpm verify` = `turbo run lint typecheck test build` + the drift check. Fix causes, never gates: no skipped tests, loosened rules, `@ts-expect-error`, or hand-edited generated files ([discipline](../../rules/testing/discipline.md)).

## Procedure

1. Run `pnpm verify` locally and identify the **first failing task** in Turbo's summary. Fix in pipeline order — an API build failure can cascade into generate, typecheck, and web build; do not chase downstream noise first.
2. Reproduce the single failing task scoped: `pnpm turbo lint --filter @repo/api`, `pnpm turbo test --filter @repo/web`, etc.
3. Classify against the table below; apply the fix; re-run the scoped task, then full `pnpm verify`.
4. If the failure is environmental (Docker down for Testcontainers, stale `node_modules`), fix the environment and note it — do not "fix" code to mask it.

## Symptom → cause → fix

| Failing task      | Symptom                                           | Likely cause → fix                                                                                                                                                              |
| ----------------- | ------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `@repo/api#lint`  | `dotnet format` reports changes                   | Unformatted C# → run `dotnet format Api.slnx` (without `--verify-no-changes`), commit                                                                                           |
| web `lint`        | ESLint boundary error                             | Deep import across features / into `*.gen.ts` → import the barrel / curated re-export                                                                                           |
| web `lint`        | `no-explicit-any`, floating promise               | Type the boundary with `unknown`+narrowing; await or void with reason                                                                                                           |
| `typecheck`       | Errors in `apps/web` referencing api-client types | Contract changed → this is the pipeline working; update adapters/components ([regenerate-api-client](../regenerate-api-client/SKILL.md)); never patch `*.gen.ts`                |
| `typecheck`       | Cannot find `@repo/api-client` types              | `generate` didn't run → `pnpm turbo run build generate`; check Turbo edge if ordering broke                                                                                     |
| `@repo/api#build` | CS errors after slice work                        | Missing nested-type wiring, wrong `DbContext`, Contracts referencing a module → fix per [slices](../../rules/dotnet-api/slices.md)/[modules](../../rules/dotnet-api/modules.md) |
| `@repo/api#build` | Warning-as-error (nullable, analyzer)             | Fix the warning; do not add `NoWarn`                                                                                                                                            |
| `@repo/api#test`  | Architecture test red                             | A boundary was crossed → move the code to the owning module/slice; the test is right                                                                                            |
| `@repo/api#test`  | Handler test can't resolve validator/handler      | Scan flags lost → `publicOnly: false` / `includeInternalTypes: true` in the module registration                                                                                 |
| `@repo/api#test`  | Integration tests fail to start container         | Docker not running / image pull → start Docker; pin `postgres:17` image                                                                                                         |
| `@repo/api#test`  | 500 where 401/403 expected                        | Missing `ErrorType.Unauthorized`/`Forbidden` mapping → model the status explicitly                                                                                              |
| web `test`        | Vitest failures on fixtures after regen           | Fixtures follow the contract → update typed fixtures from the new DTOs                                                                                                          |
| web `test`        | Flaky async test                                  | Unawaited promise / real timers → await, fake timers, no sleeps                                                                                                                 |
| drift check       | Non-empty diff on `openapi/` or `api-client/src`  | Endpoint changed without regen, or hand-edited generated file → revert hand edits, regenerate, commit outputs together                                                          |
| drift check       | Diff only in formatting/ordering of documents     | Toolchain version drift between local and CI → align via the pinning surfaces (`global.json`, lockfile), regenerate                                                             |
| several tasks     | Works locally, red in CI                          | Version skew or missing `--frozen-lockfile` sync → reinstall from lockfile, `dotnet tool restore`, compare SDK/pnpm pins                                                        |

## Escalation

- A fix that requires crossing a boundary (e.g. an architecture test "in the way") is a design question — stop and surface it; the test is the architecture.
- Repeated drift-check failures from teammates' PRs → propose the generated-file guard hook rather than repeatedly cleaning up.
- A gate that seems wrong (rule too strict, test asserting the impossible) → change it only via ADR ([architecture: ADR triggers](../../docs/architecture.md#adr-triggers)), never inline.

## Done when

- [ ] Full `pnpm verify` passes locally, cold (`--force` if caching is suspected).
- [ ] Every fix addressed a cause; no gate was weakened, skipped, or bypassed.
- [ ] Regenerated artifacts (if any) committed with the change that caused them.
- [ ] Environmental causes noted so the next person doesn't re-triage them.
