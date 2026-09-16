# Web packages and app

Pinned versions are examples — refresh patches/minors within the fixed majors at scaffold time.

## 1. Tooling packages

`packages/typescript-config/package.json`:

```json
{
  "name": "@repo/typescript-config",
  "private": true,
  "files": ["base.json", "react.json"]
}
```

`packages/typescript-config/base.json`:

```json
{
  "compilerOptions": {
    "strict": true,
    "target": "ES2023",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "verbatimModuleSyntax": true,
    "isolatedModules": true,
    "skipLibCheck": true,
    "noEmit": true,
    "forceConsistentCasingInFileNames": true
  }
}
```

`react.json` extends `base.json` with `"jsx": "react-jsx"` and DOM libs.

`packages/eslint-config` — flat config exporting `base` (TS strict rules, `no-explicit-any`, no-floating-promises) and `react` presets, plus:

- boundary rules: features import siblings only via barrels; `routes/**` imports only feature barrels; no `apps/*` imports from `packages/*`;
- read-only overrides marking `**/*.gen.ts` and `routeTree.gen.ts` as ignored for fixes and forbidden to import deeply.

## 2. packages/api-client

`package.json`:

```json
{
  "name": "@repo/api-client",
  "private": true,
  "type": "module",
  "exports": { ".": "./src/index.ts" },
  "scripts": {
    "generate": "openapi-typescript ../../apps/api/openapi/v1.json -o src/v1.gen.ts"
  },
  "dependencies": {
    "openapi-fetch": "^0.14.0"
  },
  "devDependencies": {
    "openapi-typescript": "^7.9.0",
    "@repo/typescript-config": "workspace:*"
  }
}
```

When v2 exists, `generate` chains one command per document. The package is consumed as source (`exports` → `.ts`); Vite bundles it — no build script, which is why `turbo.json` wires `@repo/web#build` to `generate` explicitly.

`src/client.ts` — config-free factory:

```ts
import createClient from 'openapi-fetch';
import type { paths as pathsV1 } from './v1.gen';

export interface ApiClientOptions {
  baseUrl: string;
  timeoutMs: number;
  headers?: () => Record<string, string>;
}

export function createApiClient({
  baseUrl,
  timeoutMs,
  headers,
}: ApiClientOptions) {
  return createClient<pathsV1>({
    baseUrl,
    fetch: (request) =>
      fetch(request, {
        signal: AbortSignal.any(
          [request.signal, AbortSignal.timeout(timeoutMs)].filter(Boolean),
        ),
      }),
    headers: headers?.(),
  });
}
```

`src/index.ts` — curated surface:

```ts
export { createApiClient, type ApiClientOptions } from './client';
export type { paths as pathsV1, components as componentsV1 } from './v1.gen';

// Stable DTO names — grown as slices appear:
// export type ProductV1 = componentsV1['schemas']['GetProductResponse'];
```

## 3. packages/ui

shadcn primitives + tokens shared across apps:

```json
{
  "name": "@repo/ui",
  "private": true,
  "type": "module",
  "exports": { "./*": "./src/*.tsx", "./styles.css": "./src/styles.css" },
  "dependencies": { "react": "^19.0.0" },
  "devDependencies": { "@repo/typescript-config": "workspace:*" }
}
```

Generic primitives only (button, dialog, input, tokens/theme in `styles.css` for Tailwind 4 `@theme`); product-aware composition stays in `apps/web` features.

## 4. apps/web

`package.json`:

```json
{
  "name": "@repo/web",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite dev --port 3000",
    "build": "vite build",
    "test": "vitest run",
    "lint": "eslint .",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "@repo/api-client": "workspace:*",
    "@repo/ui": "workspace:*",
    "@tanstack/react-query": "^5.90.0",
    "@tanstack/react-router": "^1.130.0",
    "@tanstack/react-start": "^1.130.0",
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "tailwindcss": "^4.1.0",
    "zod": "^4.0.0"
  },
  "devDependencies": {
    "@repo/eslint-config": "workspace:*",
    "@repo/typescript-config": "workspace:*",
    "@tailwindcss/vite": "^4.1.0",
    "@tanstack/react-router-plugin": "^1.130.0",
    "@testing-library/react": "^16.3.0",
    "@vitejs/plugin-react": "^5.0.0",
    "jsdom": "^26.0.0",
    "vite": "^7.0.0",
    "vitest": "^3.2.0",
    "typescript": "^5.9.0"
  }
}
```

`vite.config.ts`:

```ts
import { defineConfig } from 'vite';
import { tanstackStart } from '@tanstack/react-start/plugin/vite';
import react from '@vitejs/plugin-react';
import tailwindcss from '@tailwindcss/vite';
import tsconfigPaths from 'vite-tsconfig-paths';

export default defineConfig({
  plugins: [tsconfigPaths(), tanstackStart(), react(), tailwindcss()],
  test: {
    environment: 'jsdom',
    include: ['src/**/*.test.{ts,tsx}'],
  },
});
```

(`@` path alias → `./src` via tsconfig `paths` + `vite-tsconfig-paths`.)

`src/router.tsx` — one `QueryClient` **per request**, SSR integration installed:

```tsx
import { createRouter } from '@tanstack/react-router';
import { QueryClient } from '@tanstack/react-query';
import { setupRouterSsrQueryIntegration } from '@tanstack/react-router-ssr-query';
import { routeTree } from './routeTree.gen';

export function getRouter() {
  const queryClient = new QueryClient();

  const router = createRouter({
    routeTree,
    context: { queryClient },
    defaultPreload: 'intent',
  });

  setupRouterSsrQueryIntegration({ router, queryClient });

  return router;
}
```

`src/lib/config.ts` — validated, server-only:

```ts
import { z } from 'zod';

const schema = z.object({
  apiBaseUrl: z.string().url(),
  apiTimeoutMs: z.coerce.number().int().positive().default(5000),
});

export const serverConfig = schema.parse({
  apiBaseUrl: process.env.API_BASE_URL,
  apiTimeoutMs: process.env.API_TIMEOUT_MS,
});
```

A malformed environment fails the server boot with a named error; nothing here is importable from client-only code paths (enforced by server-only module convention + lint).

`src/lib/api-client.ts` instantiates `createApiClient(serverConfig…)` with correlation-header forwarding — full example in [add-web-feature references](../../add-web-feature/references/feature-example.md).

Scaffold `src/routes/__root.tsx` (document shell, error boundary), `src/routes/index.tsx` (placeholder), `src/features/` (empty until [add-web-feature](../../add-web-feature/SKILL.md)), `src/styles/` with Tailwind entry, and `components.json` for the shadcn CLI pointing feature-agnostic output at `packages/ui`.
