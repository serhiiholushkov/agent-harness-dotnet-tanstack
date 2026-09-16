---
title: Modules and imports
description: Extensionless relative imports (Vite bundler resolution), feature barrels, import type.
appliesTo: 'apps/web/**/*.ts, apps/web/**/*.tsx, packages/**/*.ts, packages/**/*.tsx'
---

# Modules and imports

Resolution model — TypeScript here is consumed as source and bundled by Vite (`moduleResolution: "bundler"`). There is **no** Node type-stripping runtime in this repository, so:

- Relative imports are written **without** file extensions: `import { ProductList } from './components/product-list'`. Do not add `.js`/`.ts` extensions.
- ESM syntax only: `import`/`export`, no `require`, no CommonJS interop in workspace code.

Import edges:

- Inside a feature: relative imports.
- Across features: only the sibling's public barrel — `@/features/catalog` — never a deep path into another feature's internals.
- Workspace packages by name: `@repo/api-client`, `@repo/ui`. Never a relative path that escapes the package (`../../packages/…`), and never `apps/*` source from a package.
- From `@repo/api-client`, import only the package root (curated re-exports and the client factory). Deep imports into `src/*.gen.ts` are forbidden.
- Generated files (`routeTree.gen.ts`, `*.gen.ts`) are imported, never edited.

Hygiene:

- `import type { … }` for type-only imports (`verbatimModuleSyntax`), so types are erased cleanly at build.
- Barrels (`index.ts`) exist at feature roots and package roots as public APIs — not in every subfolder.
- No import cycles between features; compose siblings in routes or app-level components ([../web/features.md](../web/features.md)).

Caught by: ESLint boundaries + import rules, `pnpm typecheck`, Vite build failures on bad resolution.
