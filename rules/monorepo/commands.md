---
title: Command surface
description: The canonical commands for developing, verifying, generating, and migrating. Use these, not ad hoc invocations.
appliesTo: '**/*'
---

# Command surface

Setup (once per clone):

```bash
pnpm install
dotnet tool restore
docker compose up -d db     # PostgreSQL; connection string from user secrets or .env
```

Daily:

| Command          | Does                                                                             |
| ---------------- | -------------------------------------------------------------------------------- |
| `pnpm dev`       | Web (`:3000`) + API (`:5000`) together via `turbo run dev`                       |
| `pnpm lint`      | ESLint/Prettier on web packages + `dotnet format --verify-no-changes` on the API |
| `pnpm typecheck` | tsc across the workspace (depends on generated client)                           |
| `pnpm test`      | Vitest + `dotnet test`                                                           |
| `pnpm build`     | All builds; API build emits `openapi/*.json`                                     |
| `pnpm generate`  | Regenerate `packages/api-client` from the OpenAPI docs                           |
| `pnpm verify`    | The full PR gate: lint, typecheck, test, build, then the drift check             |

Contract drift check (also run by CI):

```bash
pnpm turbo run build generate
git diff --exit-code -- apps/api/openapi packages/api-client/src
```

Migrations (per module, pinned tool):

```bash
pnpm --filter @repo/api exec dotnet ef migrations add <Name> \
  --project src/Modules/<Module>/<Module> \
  --startup-project src/Api \
  --context <Module>DbContext
```

Rules:

- Run commands from the repository root through pnpm/turbo so ordering and caching hold; do not bypass with raw `tsc`, `vitest`, or `dotnet` invocations except inside `apps/api` scripts where they are defined.
- `pnpm verify` green is the completion gate for any change; migrations are applied as a gated deployment step, never by `pnpm dev` or app startup.
- Use `--filter` to scope turbo runs while iterating (`pnpm turbo test --filter @repo/web`); the unfiltered gate still decides done.
