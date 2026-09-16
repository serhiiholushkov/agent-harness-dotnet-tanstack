---
name: plan-feature
description: 'Turn a feature request into a reviewable implementation plan before any code: owning module and web feature, affected layers, contract impact, data impact, ADR triggers, and a test plan. Use when starting nontrivial work, when asked to plan/estimate/break down a feature, or before touching more than one slice. Trigger terms: plan, break down, design, feature request, scope, implementation plan.'
---

# Plan a feature

Produces a plan artifact a human can approve before implementation. Planning is read-only — no code edits. Authority for every classification: [../../docs/architecture.md](../../docs/architecture.md).

## Procedure

1. **Restate the requirement** in one paragraph: actor, action, outcome, constraints. Name the unknowns explicitly instead of assuming.
2. **Assign ownership.**
   - API: which existing module owns the data and rules? If none does, that is an ADR-gated new-module decision ([add-module](../add-module/SKILL.md)) — flag it, do not default to "add a module".
   - Web: which product-concept feature folder? New folder or extension?
3. **Enumerate affected layers**, one line each — only what actually changes: API slices (per use case: new slice / new endpoint version / none), entities + domain rules, schema (migration?), Contracts (new method/event? which consumers?), cache keys, web adapters, queries, components, routes.
4. **Classify contract impact** — exactly one of:
   - none (internal change; no OpenAPI diff expected)
   - additive (new operation or optional field; same version)
   - breaking (route [version-endpoint](../version-endpoint/SKILL.md); requires a version and a migration path for web callers)
5. **Classify data impact:** none / new columns-tables in the owning schema (list migrations) / cross-module read (which table row of the read-composition table applies) / cross-module write (ADR required — stop and surface).
6. **Scan ADR triggers** ([architecture](../../docs/architecture.md#adr-triggers)): new module, new dependency, cross-module write strategy, boundary exception, new major version, shared-code promotion. List each hit with a one-line proposed decision; these are approval points, not plan steps.
7. **Write the test plan** from the test-type tables ([api-testing](../../rules/testing/api-testing.md), [web-testing](../../rules/testing/web-testing.md)): which validator/handler/integration tests per slice; which adapter/query/component tests per web change; cache and event tests where those exist.
8. **Sequence the work** as skill invocations in pipeline order, e.g. `add-ef-migration` → `add-vertical-slice` (×N) → `regenerate-api-client` → `add-web-feature` → `pnpm verify`. One vertical at a time; each step ends verifiable.
9. **Emit the plan** in this format:

```markdown
## Plan: <feature name>

**Requirement.** <one paragraph + open questions>

**Ownership.** API: <module> (existing|new+ADR). Web: <feature folder> (existing|new).

**Contract impact.** none | additive | breaking → <operations affected>

**Data impact.** <none | migrations list | cross-module read shape | cross-module write + ADR>

**ADR triggers.** <none | list with proposed decisions>

**Steps.**

1. <skill / action> — <what and where>
2. …

**Tests.** <per-layer list>

**Out of scope.** <explicitly excluded>
```

## Done when

- [ ] Every affected layer is enumerated with its owning module/feature; no step edits code outside them.
- [ ] Contract and data impact are classified in the plan's vocabulary (none/additive/breaking; read-shape table row).
- [ ] Every ADR trigger is surfaced as an approval point, not silently decided.
- [ ] The test plan names concrete test classes/files, not "add tests".
- [ ] Steps map to skills in pipeline order and end with `pnpm verify`.
- [ ] Open questions are listed rather than assumed away.
