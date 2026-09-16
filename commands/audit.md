---
name: audit
description: Run the read-only reviewers over a change set and collect their reports.
argument-hint: [diff range, default: working tree vs default branch] [--security]
---

# /audit

Delegate to reviewers over the given diff:

1. [boundary-auditor](../agents/boundary-auditor.md) — always.
2. [contract-reviewer](../agents/contract-reviewer.md) — when the diff touches `apps/api/openapi`, endpoints, or `Response` types.
3. [security-reviewer](../agents/security-reviewer.md) — with `--security`, or when the diff touches auth, configuration, or server functions.

**Inputs.** Optional diff range: `$ARGUMENTS` (default: working tree against the default branch).

**Expected report.** Each reviewer's verdict and findings table, concatenated, plus a one-line combined verdict: pass only if no reviewer failed. Findings are routed back for fixes; reviewers never edit.
