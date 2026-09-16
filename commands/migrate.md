---
name: migrate
description: Generate, review, and gate an EF Core migration for one module's schema.
argument-hint: <Module> <MigrationName> [--apply]
---

# /migrate

Activate [skills/add-ef-migration](../skills/add-ef-migration/SKILL.md).

**Inputs.** The owning module and a PascalCase migration name: `$ARGUMENTS`. Applying to a shared database is gated — only with explicit `--apply` and per the skill's apply rules.

**Expected report.** The generated migration files, the reviewed SQL summary (schema ownership confirmed, no cross-schema foreign keys, expand/contract noted where relevant), snapshot status, and whether it was applied or left pending.
