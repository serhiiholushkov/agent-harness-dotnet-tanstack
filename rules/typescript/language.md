---
title: TypeScript language discipline
description: Strict mode everywhere; no any; type errors are fixed, never silenced.
appliesTo: 'apps/web/**/*.ts, apps/web/**/*.tsx, packages/**/*.ts, packages/**/*.tsx'
---

# TypeScript language discipline

- `strict: true` in every tsconfig, extended from `@repo/typescript-config`. Per-package overrides may tighten, never loosen.
- `any` is forbidden — explicit or implicit. Unknown input is `unknown`, narrowed with type guards or schema validation at the boundary where it enters.
- `@ts-ignore`, `@ts-expect-error`, and `as any` are not fixes; they are gate-weakening ([../testing/discipline.md](../testing/discipline.md)). The rare justified `@ts-expect-error` carries a reason and a link to the upstream issue.
- Non-null assertions (`!`) are forbidden in application code; narrow or handle the absent case. Type assertions (`as T`) only where TypeScript provably cannot know better (e.g. after schema validation), never to silence a mismatch.
- Prefer `const`; no `var`. Functions that are exported from a feature barrel or package have explicit return types when inference would expose internals.
- Type errors surface via `pnpm typecheck` (Vite does not typecheck); code is not done while typecheck fails.

Caught by: `pnpm typecheck` (tsc across the workspace), ESLint (`no-explicit-any`, restricted comments), review.
