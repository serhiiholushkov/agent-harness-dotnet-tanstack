---
name: implement
description: Implement a planned feature as a full vertical — API slice(s), regenerated client, web feature — in pipeline order.
argument-hint: <plan reference or feature description> [--api-only | --web-only]
---

# /implement

Activate, in pipeline order:

1. [skills/add-vertical-slice](../skills/add-vertical-slice/SKILL.md) — per use case in the owning module (skip with `--web-only`).
2. [skills/regenerate-api-client](../skills/regenerate-api-client/SKILL.md) — when the contract changed.
3. [skills/add-web-feature](../skills/add-web-feature/SKILL.md) — the web side (skip with `--api-only`).

**Inputs.** An approved plan from [/plan](plan.md), or a request small enough to need none: `$ARGUMENTS`. Unplanned ADR-gated decisions abort — route back to [/plan](plan.md).

**Expected report.** Files added/changed per skill, the reviewed `openapi/*.json` diff summary, tests added, and the `pnpm verify` result. Each skill's "done when" checklist satisfied.
