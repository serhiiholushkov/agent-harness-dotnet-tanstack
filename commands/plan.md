---
name: plan
description: Produce a reviewable implementation plan for a feature request before any code is written.
argument-hint: <feature request or issue reference>
---

# /plan

Delegate to the [planner](../agents/planner.md) agent, which executes [skills/plan-feature](../skills/plan-feature/SKILL.md).

**Inputs.** The feature request verbatim: `$ARGUMENTS`. If it names no acceptance criteria, the planner records them as open questions.

**Expected report.** The `## Plan: <feature name>` artifact — ownership, contract and data impact, ADR triggers as approval points, sequenced skill steps, test plan. No code edits.

**Follow-up routing.** Approved plan steps run via [/implement](implement.md), [/add-module](add-module.md), [/migrate](migrate.md), or [/version-endpoint](version-endpoint.md) as the plan names them.
