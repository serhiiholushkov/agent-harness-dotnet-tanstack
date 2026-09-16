---
title: Testing discipline
description: Tests are the gate; never weaken them to pass. What tests must and must not do.
appliesTo: 'apps/api/tests/**, apps/web/**/*.test.ts, apps/web/**/*.test.tsx'
---

# Testing discipline

Never:

- Skip, quarantine, or delete a failing test to go green. A failing test is either a real bug (fix the code) or a wrong expectation (fix the test with justification in the commit message).
- Loosen an assertion, widen a type, add `@ts-expect-error`, disable a lint rule, or mark `[Fact(Skip=…)]` as a shortcut to a passing gate.
- Assert on implementation details — private state, call counts that do not matter, exact log strings — instead of observable behavior (responses, DOM, persisted rows, published events).
- Leave state behind: every test closes what it opens (containers, files, fake timers, spies). Web tests reset the `QueryClient`; API integration tests own their data lifecycle.
- Depend on test order, wall-clock time (`IDateTimeProvider` / fake timers exist for this), or the network beyond the Testcontainers instance.

Always:

- Add a regression test with every bug fix — it fails before the fix and passes after.
- Test every `Result.Failure` branch, every adapter error path, every ownership rule. Happy-path-only coverage is incomplete by definition.
- Keep fakes typed by the real interfaces (Contracts on the API, generated DTOs on the web) so drift is a compile error.
- Keep end-to-end tests only for critical cross-app workflows; the pyramid here is validator/handler/component unit tests → module integration → few E2E.
- Treat flakiness as a defect: deterministic seeds, awaited async, no sleeps.

Generated files need no tests; migrations are covered by integration tests applying them.

Caught by: review, CI history (flaky-test tracking), the completion gate refusing skipped suites.
