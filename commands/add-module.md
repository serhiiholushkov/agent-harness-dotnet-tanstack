---
name: add-module
description: Create a new business module in the API — ADR-gated; requires an approved ADR before any code.
argument-hint: <ModuleName> — <one-line capability it owns>
---

# /add-module

Activate [skills/add-module](../skills/add-module/SKILL.md).

**Inputs.** Module name and owned capability: `$ARGUMENTS`, plus the approving ADR reference. Without an ADR, stop and emit the decision for approval — creating a module is an ADR trigger, never a default.

**Expected report.** Projects created (`{Module}.Contracts`, `{Module}`), schema/DbContext/migrations-history names, registration wiring, isolation architecture tests added, and the `pnpm verify` result.
