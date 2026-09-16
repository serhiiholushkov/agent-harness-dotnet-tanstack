---
name: verify
description: Run the full PR gate and, if red, triage per the fix-verify-failures skill.
argument-hint: [--scope <package>]
---

# /verify

Run `pnpm verify` (with `--scope`, the scoped Turbo equivalent first, then the full gate). On failure, activate [skills/fix-verify-failures](../skills/fix-verify-failures/SKILL.md).

**Inputs.** None required.

**Expected report.** Green: the passing task list. Red: the first failing task, its classified cause from the skill's symptom table, the fix applied (never a weakened gate), and the rerun result. Environmental causes are reported as such, not patched into code.
