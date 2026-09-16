---
title: Data ownership
description: Schema-per-module, migrations ownership, and the lawful shapes for cross-module data access.
appliesTo: 'apps/api/src/**/*.cs'
---

# Data ownership

Ownership:

- One database; one schema per module (`HasDefaultSchema(Schemas.{Module})`); one `internal sealed` `DbContext` per module with its own migrations history table in its own schema.
- A module is the only code that queries its schema. No cross-schema foreign keys and no cross-schema joins — including in raw SQL, where no compiler can see it. External identifiers are plain `uuid` columns.
- Migrations are generated per module (command surface: [../monorepo/commands.md](../monorepo/commands.md)) and applied as a reviewed, gated deployment step — never at startup.
- A new module's first migration must add no foreign key crossing out of its schema.

Cross-module reads — pick by requirement, never a join:

| Requirement                            | Required shape                                                                  |
| -------------------------------------- | ------------------------------------------------------------------------------- |
| One record + current sibling details   | One Contracts call in the query handler, compose in memory                      |
| A page enriched with sibling details   | Page locally, dedupe ids, **one** batch Contracts call per page, join in memory |
| Sort/filter/page by a sibling field    | Event-fed projection in the consumer's schema; query only the local table       |
| Historical value (price at order time) | Snapshot during the write; never updated when the source changes                |
| Dashboard across modules               | Reporting module with its own event-fed projections                             |

- The owning module implements batch APIs with one `WHERE Id IN (...)` query; consumers never issue per-row calls. A missing summary stays `null` unless the use case must fail.
- Contracts APIs are in-process DI interfaces — never HTTP loopback, never the generated TypeScript client inside the API.

Cross-module writes and events:

- One command, one module, one `SaveChangesAsync`. Any cross-module write workflow explicitly chooses shared transaction, compensation, or reserve-then-confirm — an ADR-gated decision.
- Integration events go through the module's **outbox** (same transaction as the state change, via interceptor) and the consumer's **inbox** (keyed by event id, processed once). Inline publishing loses events under load.
- Projections and inbox consumers write only into the consuming module's schema.

Caught by: migration review (FKs, schemas), NetArchTest isolation tests, cross-module integration tests incl. outbox/inbox delivery, review of raw SQL.
