---
name: add-ef-migration
description: "Generate, review, and gate an EF Core migration for one module's PostgreSQL schema. Use when an entity, configuration, or index changes, when adding a module's first migration, or when asked to change the database schema. Trigger terms: migration, dotnet ef, schema change, add column, add index, alter table, migrations history."
---

# Add an EF Core migration

Schema changes are per module and reviewed like code. Standards: [ef-core](../../rules/csharp/ef-core.md), [data-ownership](../../rules/dotnet-api/data-ownership.md), [commands](../../rules/monorepo/commands.md).

## Decision points

- **Which module owns the change?** The migration is generated against that module's `DbContext` and lands in its `Database/Migrations`. A change spanning two schemas is two migrations — and a design smell to question first.
- **Is the change destructive or breaking for running instances?** Renames, type changes, `NOT NULL` on existing columns, and drops require **expand/contract**: add the new shape (expand), migrate data and switch code, drop the old shape in a later migration (contract). Never rely on a single big-bang migration for zero-downtime deploys.
- **New module?** The first migration also creates the schema and the module's own migrations history table — that comes from the `MigrationsHistoryTable` setup, not from hand-editing.

## Procedure

1. Create or change the entity and its `IEntityTypeConfiguration<T>` under the module's `Database/Configurations/` — the entity and this migration are one change set, committed together. Constraints (uniqueness, in-schema FKs, concurrency tokens) belong in configuration, enforced by the database.
2. Generate with the pinned tool from the repo root (name in PascalCase, states intent):

   ```bash
   pnpm --filter @repo/api exec dotnet ef migrations add AddShipmentTrackingIndex \
     --project src/Modules/Shipping/Shipping \
     --startup-project src/Api \
     --context ShippingDbContext
   ```

3. Review the generated migration — the checklist:
   - Every object lives in the module's own schema; **no foreign key or reference crosses schemas** (external ids stay plain `uuid`).
   - Table/column names are snake_case (the naming convention did it; suspicious PascalCase means a raw string sneaked in).
   - Destructive operations are staged expand/contract; data migrations are explicit `migrationBuilder.Sql` with a reviewed statement, not implicit column rewrites.
   - Defaults, nullability, and index direction match the configuration intent; no accidental drops from a renamed property (EF sees rename as drop+add — use `RenameColumn`).
   - The model snapshot diff contains only this change.
4. Never edit an **applied** migration or the snapshot by hand. To fix an unapplied migration: `dotnet ef migrations remove` (same `--project`/`--context` flags), correct the model, regenerate.
5. Apply locally by running the module's integration tests — they run real migrations against Testcontainers PostgreSQL — and/or against the local compose database via `dotnet ef database update` with the same flags. **Deployment applies migrations as a reviewed, gated step (script or bundle), never at app startup.**
6. Commit the migration together with the entity/slice change that motivated it.

## Example

Adding `TrackingNumber` to `Shipment`: property on the entity (`private set`), `builder.Property(s => s.TrackingNumber).HasMaxLength(64)` + unique index in `ShipmentConfiguration`, generate `AddShipmentTrackingNumber`, review that it emits one `ADD COLUMN` + `CREATE UNIQUE INDEX` in schema `shipping`, run integration tests, commit with the slice using it.

## Done when

- [ ] Migration generated with the canonical per-module command; lands in the owning module's `Database/Migrations`.
- [ ] Review checklist passed — own schema only, no cross-schema FK, staged destructive changes, clean snapshot diff.
- [ ] Integration tests (real migrations on Testcontainers) pass; no `Database.Migrate()` added to startup.
- [ ] Migration committed with the motivating change; `pnpm verify` passes.
