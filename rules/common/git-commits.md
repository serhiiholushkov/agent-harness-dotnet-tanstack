---
title: Git commits and PRs
description: Conventional Commits, contract-diff hygiene, and pull request expectations.
appliesTo: '**/*'
---

# Git commits and PRs

Commits:

- Follow Conventional Commits: `type(scope): subject`.
  - Types: `feat`, `fix`, `refactor`, `test`, `docs`, `chore`, `build`, `ci`, `perf`.
  - Scope is the module, feature, or package touched: `feat(catalog): …`, `fix(web/checkout): …`, `chore(api-client): …`.
  - Subject: imperative, lower case, no trailing period.
- Body explains why, not what. Breaking API contract changes carry a `BREAKING CHANGE:` footer naming the affected operation ids.
- An endpoint change and its regenerated artifacts (`apps/api/openapi/*.json`, `packages/api-client/src/*.gen.ts`) are committed together — never in separate commits that leave the tree drifted.
- Migrations are committed with the slice/entity change that motivated them.
- Never commit secrets, `.env` files, or hand edits to generated files.

Pull requests:

- One vertical or one concern per PR: a slice plus its web feature is one PR; unrelated refactoring is not.
- The PR description calls out the `openapi/*.json` diff when endpoints changed — that diff is the contract change and the primary review artifact.
- `pnpm verify` is green before review is requested. Do not use `--no-verify` or skip hooks to get there.
- New behavior arrives with its tests; bug fixes arrive with a regression test.
- ADR-gated decisions (see the ADR Triggers section of [../../docs/architecture.md](../../docs/architecture.md#adr-triggers)) link the ADR.

Caught by: commit-lint style review, CI running the full gate and drift check, reviewers.
