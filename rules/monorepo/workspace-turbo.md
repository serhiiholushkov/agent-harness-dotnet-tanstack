---
title: Workspace and Turbo
description: pnpm workspace shape and the Turbo task graph rules the pipeline depends on.
appliesTo: 'turbo.json, package.json, pnpm-workspace.yaml, apps/*/package.json, packages/*/package.json'
---

# Workspace and Turbo

Workspace:

- pnpm workspaces own dependency management; Turborepo 2 owns task ordering, filtering, and caching for **both** toolchains. MSBuild owns C# compilation.
- `apps/api/package.json` is a thin wrapper — `dotnet` invocations only, no JavaScript, no build logic. Its scripts are `dev`/`build`/`test`/`lint` shelling to `dotnet watch run`, `dotnet build Api.slnx`, `dotnet test`, `dotnet format --verify-no-changes`.
- Pins: `packageManager` (pnpm) in the root `package.json`, `global.json` for the .NET SDK, `.config/dotnet-tools.json` for `dotnet-ef`, `Directory.Packages.props` for NuGet. Version changes touch the pin, never an individual project.

Turbo task rules (the four the pipeline depends on):

1. Never put `bin/` or `obj/` in a shared Turbo cache — they contain absolute paths. `@repo/api#build` sets `cache: false` and declares only `openapi/*.json` as outputs; MSBuild is already incrementally correct.
2. Never cache a task whose real output is a database or a running process (`test` against containers, `dev` is `persistent: true, cache: false`).
3. `inputs` globs are package-relative; a task reaching outside its package uses the `$TURBO_ROOT$` prefix — `@repo/api-client#generate` declares `"$TURBO_ROOT$/apps/api/openapi/*.json"`. A `../../` glob is not hashed reliably.
4. `@repo/web#build` and root `typecheck` declare `@repo/api-client#generate` explicitly — the client is consumed as source, has no `build` script, so `^build` alone never orders codegen first.

Also:

- `globalDependencies` include `global.json` and `.config/dotnet-tools.json` so toolchain changes bust caches.
- Root `package#task` overrides **replace** the base task config, they do not merge — re-declare everything the override needs.
- New tasks declare honest `inputs`/`outputs`; an undeclared output is a silent cache-poisoning bug.

Caught by: drift check and CI cold-cache runs, review of `turbo.json` diffs.
