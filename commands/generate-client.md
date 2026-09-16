---
name: generate-client
description: Regenerate the OpenAPI documents and TypeScript client, and review the contract diff.
argument-hint: [--check-only]
---

# /generate-client

Activate [skills/regenerate-api-client](../skills/regenerate-api-client/SKILL.md).

**Inputs.** None required. `--check-only` runs the drift check without committing: `pnpm turbo run build generate && git diff --exit-code -- apps/api/openapi packages/api-client/src`.

**Expected report.** The `openapi/*.json` diff summarized as a contract change (none/additive/breaking, per operation), regenerated `packages/api-client/src` status, curated re-export updates, and drift-check result. Hand edits to generated files are never part of the outcome.
