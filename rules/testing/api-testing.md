---
title: API testing
description: Test types, placement mirroring Features/, and what each layer must prove.
appliesTo: 'apps/api/tests/**/*.cs'
---

# API testing

Placement and naming:

- Test projects mirror `Features/` slice by slice: `tests/Modules/Catalog/Catalog.UnitTests/Features/Products/GetProductHandlerTests.cs`. Classes are named after the slice — `GetProductHandlerTests`, `CreateProductValidatorTests` — never `ProductServiceTests`.
- Stack: xUnit + Shouldly + NSubstitute + NetArchTest + Testcontainers.PostgreSql. `InternalsVisibleTo` only to the module's own test projects.

The test-type table — every row must stay covered:

| Test type          | Proves                                                           | Dependencies                                |
| ------------------ | ---------------------------------------------------------------- | ------------------------------------------- |
| Validator          | Input rules, one test per rule                                   | None                                        |
| Handler unit       | Every `Result.Failure` branch plus the happy path                | NSubstitute doubles for ports and Contracts |
| Contract           | The module's public API returns what Contracts promises          | The module's real slices, test database     |
| Architecture       | Module isolation, slice isolation, Contracts purity, host purity | Assembly scanning                           |
| Module integration | One module's HTTP surface against its own schema                 | Real host + Testcontainers PostgreSQL       |
| Cross-module       | Flows spanning modules, incl. outbox/inbox delivery              | Real host + real database + real bus        |
| OpenAPI            | Documents generate, operation ids unique, routes present once    | Real host                                   |

Rules:

- Handler tests construct the nested handler directly and bypass decorators — so each module carries its own validation/logging pipeline coverage; a module can be silently undecorated while others are fine.
- Fakes are typed by the real Contracts interfaces (`Substitute.For<ICatalogApi>()`). If faking a contract is painful, the contract is too big — a design signal.
- Integration tests boot the real host, run **real migrations** against Testcontainers PostgreSQL (never the in-memory provider for persistence claims), and close what they open. One container serves all module schemas.
- Ownership denial is tested per slice: another principal's resource yields the same 404/403 the contract promises.
- Each cached read has a test mutating through every write path; outbox/inbox tests assert exactly-once consumption across a processor restart.
- Every slice covers at least: success, validation failure, missing resource, ownership denial, state conflict, and the HTTP status/body contract once in an integration test.

Caught by: CI running `pnpm test`; the architecture tests themselves are the enforcement for boundary rules.
