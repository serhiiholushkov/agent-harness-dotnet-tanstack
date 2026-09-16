---
name: scaffold-workspace
description: 'Bootstrap the whole monorepo from an empty folder: root pins and workspace files, Turbo wiring, the .NET solution with Common.* projects and the Api host, the web tooling packages, api-client codegen package, the TanStack Start app, Docker Compose for PostgreSQL, and CI. Use when creating the repository, initializing the project, or asked to set up the workspace/skeleton. Trigger terms: scaffold, bootstrap, new repo, init workspace, project setup, monorepo skeleton.'
---

# Scaffold the workspace

Builds the repository skeleton in verifiable stages. Every file's exact content lives in the references; this body is the order and the checks. Version numbers shown in references are pinned examples — pin to the current LTS/stable at scaffold time, but never change the majors fixed by the architecture (.NET 10, Node 24, React 19, TanStack Start 1, Tailwind 4, Turborepo 2).

## Procedure

1. **Root files** — [references/root-files.md](references/root-files.md): root `package.json` (with `packageManager` pin and the `dev|lint|typecheck|test|build|generate|verify` scripts), `pnpm-workspace.yaml`, `turbo.json` (the codegen edge and caching rules), `global.json`, `.config/dotnet-tools.json`, `.gitignore`, `.env.example`, `docker-compose.yml` (PostgreSQL on 5432).
   _Check:_ `pnpm install` succeeds; `dotnet tool restore` succeeds; `docker compose up -d db` starts PostgreSQL.
2. **Web tooling packages** — [references/web-packages.md](references/web-packages.md) §1: `packages/typescript-config` (strict, bundler resolution) and `packages/eslint-config` (boundaries, generated-file read-only overrides).
   _Check:_ both resolve as workspace deps.
3. **API solution** — [references/api-solution.md](references/api-solution.md): `apps/api` with `Api.slnx`, `Directory.Build.props`, `Directory.Packages.props`, four `Common.*` projects, the `Api` host (Program.cs, `launchSettings.json` on port 5000, build-time OpenAPI into `apps/api/openapi`), the thin `package.json` wrapper, and test projects. Seed `Common.*` with the kernel primitives — [references/common-kernel.md](references/common-kernel.md): `Result`/`Error`/`ErrorType`, messaging interfaces, decorators, `IEndpoint` + discovery, `CustomResults`, `Tags`/`ApiVersions`, schema-reference-id config, global exception handler, health checks.
   _Check:_ `pnpm --filter @repo/api build` compiles and writes `apps/api/openapi/v1.json`; `pnpm --filter @repo/api dev` serves `/health/live` on `:5000`.
4. **api-client and ui packages** — [references/web-packages.md](references/web-packages.md) §2–3: `packages/api-client` (`generate` script running `openapi-typescript` per version document, config-free `client.ts` factory, curated `index.ts`) and `packages/ui` (shadcn primitives + tokens).
   _Check:_ `pnpm turbo run build generate` produces `packages/api-client/src/v1.gen.ts`.
5. **Web app** — [references/web-packages.md](references/web-packages.md) §4: `apps/web` TanStack Start app on port 3000 — Vite config, router with per-request `QueryClient` + SSR Query integration, `src/routes/`, `src/features/`, `lib/` (validated server-only config, `api-client` instantiation), Tailwind 4 + `components.json`, Vitest setup.
   _Check:_ `pnpm dev` serves `:3000` and the app reaches the API through a server function.
6. **CI** — [references/ci.md](references/ci.md): the workflow running install, restore, `pnpm verify` (lint, typecheck, test, build, drift check) with pnpm and .NET caching and a PostgreSQL service/Testcontainers support.
   _Check:_ workflow file lints (`act` or a draft PR).
7. **First module** — do **not** invent one here. Run [add-module](../add-module/SKILL.md) for the first real business module, then [add-vertical-slice](../add-vertical-slice/SKILL.md) and [add-web-feature](../add-web-feature/SKILL.md) for the first vertical.
8. **Full gate** — `pnpm verify` passes end to end on the skeleton (with zero modules the OpenAPI documents still generate from the host's registered versions).

## Done when

- [ ] A fresh clone needs only `pnpm install`, `dotnet tool restore`, `docker compose up -d db`, `pnpm dev` to run web (`:3000`) + API (`:5000`).
- [ ] All five pinning surfaces exist and agree: `global.json`, `packageManager`, `Directory.Packages.props`, `.config/dotnet-tools.json`, `pnpm-lock.yaml`.
- [ ] `turbo.json` encodes: `@repo/api#build` uncached with `openapi/*.json` outputs; `@repo/api-client#generate` depending on it with `$TURBO_ROOT$` inputs; `@repo/web#build` + `typecheck` depending on generate; `dev` persistent.
- [ ] The drift check runs clean on the skeleton; generated files are git-tracked.
- [ ] CI runs `pnpm verify` green.
- [ ] No business module or product code was invented — the skeleton is product-agnostic.
