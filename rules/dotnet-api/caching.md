---
title: Caching
description: HybridCache with module-scoped keys and save-then-invalidate discipline.
appliesTo: 'apps/api/src/**/*.cs'
---

# Caching

- Cache inside the query handler that owns the read, through `HybridCache` (`GetOrCreateAsync`). No caching in endpoints, decorators, or Contracts implementations.
- Every key for a module lives in one `{Module}CacheKeys` static class so read/write pairs are greppable. No inline key strings in handlers.
- Keys are scoped to the module and to the user or tenant when the data is principal-specific: `catalog-product-{id}`, `orders-{userId}-{orderId}`. Two modules caching `user-{id}` in one process will collide — the symptom is one module deserializing the other's type.
- Invalidate (`RemoveAsync`) only **after** a successful `SaveChangesAsync`, never before — an invalidation ahead of a failed transaction repopulates the cache with stale data.
- Every write path that affects a cached read invalidates it — including cross-module effects, where the invalidation lives in the consuming module's integration-event handler, next to the read it protects.
- Each cached read gets an integration test that mutates through **each** write path (including the event path) and asserts the read changed.
- `AddHybridCache()` alone is in-process. More than one instance requires a configured distributed secondary store; record the choice.
- Pass the `CancellationToken` into every cache operation.

Caught by: the per-write-path integration tests, review of new write slices against `{Module}CacheKeys`.
