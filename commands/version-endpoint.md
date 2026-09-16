---
name: version-endpoint
description: Make a breaking change to a released endpoint by adding a versioned successor and deprecating the old one.
argument-hint: <operation id or route> — <what must change and why>
---

# /version-endpoint

Activate [skills/version-endpoint](../skills/version-endpoint/SKILL.md).

**Inputs.** The affected operation and the intended change: `$ARGUMENTS`. The skill's decision tree first confirms the change is actually breaking — additive changes exit to [/implement](implement.md). A new major version and the old version's retirement schedule are ADR-gated: surface before building.

**Expected report.** Breaking-or-not verdict with reasoning, the new endpoint/slice and its version group, OpenAPI document changes, deprecation metadata on the old operation, regenerated client diff summary, and the `pnpm verify` result.
