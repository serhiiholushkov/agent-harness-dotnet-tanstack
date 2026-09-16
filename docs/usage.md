# Usage guide — driving the framework

How humans and agents get value out of this catalog: what loads when, how to phrase work, what to demand back, and where agents typically go wrong in this stack.

## Component roles

A fact lives in exactly one place; everything else links to it.

| Kind      | Job                                                     | Loaded                        | Lives in                              |
| --------- | ------------------------------------------------------- | ----------------------------- | ------------------------------------- |
| Contract  | The always-on digest: precedence, stack, commands, gate | Always                        | [../AGENTS.md](../AGENTS.md)          |
| Rule      | A durable standard the code must always meet            | Always (per installed pack)   | [../rules/](../rules/README.md)       |
| Skill     | A procedure for one kind of task                        | When the task matches         | [../skills/](../skills/README.md)     |
| Agent     | A scoped worker (planning or review) with limited tools | When delegated                | [../agents/](../agents/README.md)     |
| Command   | A user entry point that routes to a skill/agent         | When invoked                  | [../commands/](../commands/README.md) |
| Hook      | A deterministic check outside the model                 | On harness/git events         | [../hooks/](../hooks/README.md)       |
| Authority | The complete normative architecture                     | On demand; wins all conflicts | [architecture.md](architecture.md)    |

What this means in practice: the agent always has the contract and the installed rule packs in context; it pulls a skill in when the task matches its description; you (or an orchestrating agent) delegate reviews to agents and trigger workflows through commands; hooks and CI catch what slips through regardless.

## Driving agents — what works in this stack

**State the boundary, not the code.** "Add `GET /products/{id}` to the Catalog module as a slice with a cached read" beats a paragraph of implementation detail. The rules and the [add-vertical-slice](../skills/add-vertical-slice/SKILL.md) skill already encode _how_; your job is _what_ and _where it belongs_.

**Point at the neighbour.** The strongest instruction in a conventions-heavy codebase is "make it look like `GetProduct.cs`". One correct existing slice, feature folder, or module outperforms restating the rules.

**One vertical at a time.** API slice → `pnpm generate` → web feature → `pnpm verify`, then the next use case. Batching several use cases across both toolchains multiplies the blast radius of every mistake and makes the contract diff unreadable.

**Plan before multi-slice work.** Route anything nontrivial through [/plan](../commands/plan.md) first and approve the plan artifact — cheap to correct at that stage, expensive after five files changed. The plan names ADR triggers before they get silently decided.

**Demand the two artifacts of proof.** For any change that touched the API: the `apps/api/openapi/*.json` diff (the contract change, human-readable) and the `pnpm verify` output. An agent that cannot produce both is not done. For review beyond that, run [/audit](../commands/audit.md).

**Feed failures back precisely.** Paste the first failing task's output, not "it's broken". [fix-verify-failures](../skills/fix-verify-failures/SKILL.md) maps symptoms to causes; agents triage well when given the actual error and badly when guessing.

## Stack-specific failure modes and countermeasures

| Failure mode                                                                     | Countermeasure                                                                                                                              |
| -------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| Editing `*.gen.ts` / `openapi/*.json` to "fix" a type error                      | The [generated-file guard](../hooks/README.md) blocks it; the fix is always in C# + `pnpm generate`                                         |
| Hand-writing TypeScript DTOs instead of importing `@repo/api-client`             | Rule [typescript/types](../rules/typescript/types.md); reviewers flag it; ask "where does this type come from?"                             |
| Calling the API from the browser or leaking `API_BASE_URL` client-side           | Rule [web/server-functions](../rules/web/server-functions.md); [security-reviewer](../agents/security-reviewer.md)                          |
| Cross-module shortcuts: referencing another module's project, cross-schema joins | Architecture tests fail the build; [boundary-auditor](../agents/boundary-auditor.md) cites the edge                                         |
| Per-row Contracts calls in a loop to compose a page                              | [cross-module-communication](../skills/cross-module-communication/SKILL.md) owns the lawful read shapes                                     |
| "Fixing" a red gate by skipping tests, `NoWarn`, or `@ts-expect-error`           | [testing/discipline](../rules/testing/discipline.md); weakening a gate is an ADR trigger, never inline                                      |
| Editing an applied migration to change the schema                                | Guard + [add-ef-migration](../skills/add-ef-migration/SKILL.md): schema changes are always a _new_ migration                                |
| Silent breaking contract changes inside v1                                       | [contract-reviewer](../agents/contract-reviewer.md) classifies the diff; breaking → [version-endpoint](../skills/version-endpoint/SKILL.md) |
| Endpoint metadata shortcuts (`Results.Ok` untyped, missing group name)           | Boot test + generated `unknown` types make it visible; rule [dotnet-api/endpoints-openapi](../rules/dotnet-api/endpoints-openapi.md)        |
| Scope creep: refactoring beyond the task "while at it"                           | The contract's conduct section; plans list "out of scope"; reviewers flag unrelated diffs                                                   |

## What agents must never decide alone

The full list is [architecture.md — ADR Triggers](architecture.md#adr-triggers); the ones agents hit most:

- Creating, merging, or splitting a module — surfacing it is [/add-module](../commands/add-module.md)'s job; deciding it is yours.
- Adding any dependency, tool, or pattern outside the Fixed Stack.
- A cross-module write strategy (shared transaction, compensation, reserve-then-confirm).
- A new major API version and the old version's retirement schedule.
- Any exception to a boundary rule, however temporary it sounds.
- Weakening or disabling a verification gate.

An agent behaving correctly stops at these and presents options with trade-offs. Approve by recording an ADR ([templates/adr.template.md](../templates/adr.template.md)), then let it proceed.

## Session shapes that work

- **Feature**: `/plan` → approve (record ADRs if flagged) → `/implement` → `/audit` → `/verify` → PR with the contract diff in the description.
- **Schema change**: `/migrate Catalog AddProductDimensions` → review the SQL summary → apply per the skill's gate.
- **Breaking change**: `/version-endpoint` → confirm the breaking verdict → build v2 beside v1 → migrate web adapters → deprecation plan as an ADR.
- **Red CI**: paste the failing output → `/verify` → fix-verify-failures triage → never accept a fix that weakens a gate.
- **Bootstrap**: give an empty repo and [scaffold-workspace](../skills/scaffold-workspace/SKILL.md); verify with `pnpm verify` plus the drift check before writing any feature.
