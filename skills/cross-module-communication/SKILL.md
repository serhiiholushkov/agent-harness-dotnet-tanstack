---
name: cross-module-communication
description: "Wire one API module to another the lawful way: Contracts interfaces for synchronous reads, batch APIs for pages, outbox/inbox integration events for reactions and projections, snapshots for historical values, and explicit strategies for cross-module writes. Use when a slice needs another module's data, must react to another module's change, or a workflow spans modules. Trigger terms: cross-module, Contracts, integration event, outbox, inbox, projection, batch read, snapshot, saga, consistency."
---

# Cross-module communication

Modules talk only through Contracts (synchronous) or integration events (asynchronous). Everything else — sibling `DbContext`, handler calls, cross-schema SQL, HTTP loopback — is forbidden ([data-ownership](../../rules/dotnet-api/data-ownership.md), [modules](../../rules/dotnet-api/modules.md)).

## Decision points

| Situation                                          | Mechanism                                     |
| -------------------------------------------------- | --------------------------------------------- |
| The slice needs data to make a decision            | Contract interface                            |
| The slice must fail if the other module fails      | Contract interface                            |
| Another module reacts to something that happened   | Integration event                             |
| The reaction may be delayed, retried, or reordered | Integration event                             |
| Sort/filter/page by a sibling-owned field          | Event-fed projection in the consumer's schema |
| Historical value (price at order time)             | Snapshot during the write                     |
| Two modules must change together atomically        | Neither — the boundary is wrong; escalate     |

First ask whether the call is needed at all — a token that already proves a user exists makes an existence check redundant. The cheapest cross-module call is the one you delete.

## Procedure A — synchronous read via Contracts

1. In the **owning** module's Contracts, add the minimal method: single lookup `GetAsync(id, ct)` returning `Summary?`, or for pages a batch `GetManyAsync(ids, ct)` returning `IReadOnlyDictionary<Guid, Summary>`. Purpose-built DTOs only; adding a method is a contract change with an owner — push back on one-method-per-question growth.
2. Implement it in the owning module's `PublicApi/`, going through the module's own slices, one `WHERE Id IN (...)` query for batches. Return `null` / partial dictionaries, never the module's errors.
3. In the **consuming** slice, inject the interface, call it **once per request** (dedupe ids for pages, join in memory). A missing summary stays `null` in the read model unless the use case must fail.
4. Test: consumer handler with `Substitute.For<ICatalogApi>()`; owning side gets a contract test proving the promise against a real database.

Worked shape (Orders composing Catalog data): [references/examples.md](references/examples.md).

## Procedure B — asynchronous reaction via integration events

1. Define the event in the **owning** module's Contracts (`ProductPriceChangedIntegrationEvent`). Events are versioned public data: additive changes only, never rename/repurpose properties once consumed.
2. The owning slice raises a domain event on the entity; the outbox interceptor persists it in the same transaction; the outbox processor publishes the integration event. No inline publishing from slices.
3. The **reacting** module consumes through its inbox (keyed by event id, processed once) with the consumer beside the reacting feature: `Orders/Features/Orders/OnProductPriceChanged.cs`. It writes only into its own schema — projection rows, cache invalidation, follow-up commands.
4. For projections: create the local table via [add-ef-migration](../add-ef-migration/SKILL.md), populate from events, and query only that table for sort/filter/page needs. Accept eventual consistency deliberately.
5. Test: outbox/inbox integration test — publish, kill the processor mid-flight, restart, assert exactly-once at the consumer.

## Procedure C — cross-module writes

There is no cross-module transaction by construction. If a workflow must write to two modules, choose explicitly and record an ADR ([architecture: transactions](../../docs/architecture.md#transactions-across-modules)): shared transaction (couples extraction), compensating action (needs durable idempotent identities), or reserve-then-confirm (most work, crash-safe). If most slices in a module need this, the module boundary is wrong — say so instead of implementing around it.

## Done when

- [ ] The mechanism matches the decision table; no forbidden edge introduced.
- [ ] Contracts changes are minimal, purpose-built, and reviewed as public API; batch reads are one call per page.
- [ ] Events flow outbox → bus → inbox; consumers are idempotent and write only their own schema.
- [ ] Consumer fakes the contract in unit tests; owning module has a contract test; event flows have an exactly-once integration test.
- [ ] Any cross-module write strategy is ADR-recorded.
- [ ] `pnpm verify` passes.
