# CI workflow

GitHub Actions is the reference; the required jobs translate to any CI. The gate is exactly `pnpm verify` — CI must not invent a different command path than developers run.

`.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v4 # reads packageManager from package.json

      - uses: actions/setup-node@v4
        with:
          node-version: 24
          cache: pnpm

      - uses: actions/setup-dotnet@v4
        with:
          global-json-file: global.json

      - name: Restore
        run: |
          pnpm install --frozen-lockfile
          dotnet tool restore

      - name: Verify (lint, typecheck, test, build, drift check)
        run: pnpm verify
        env:
          # Testcontainers uses the runner's Docker daemon for PostgreSQL
          TESTCONTAINERS_RYUK_DISABLED: 'false'
```

Notes:

- Integration tests provision PostgreSQL via Testcontainers on the runner's Docker daemon — no service container needed; keep one, instead, only if Testcontainers is unavailable in the environment.
- The drift check inside `verify` fails the build on a stale or hand-edited contract (`git diff --exit-code -- apps/api/openapi packages/api-client/src`) — this is rule 7 of the architecture enforced mechanically.
- `--frozen-lockfile` keeps CI honest about `pnpm-lock.yaml`; a lockfile change belongs in the PR.
- NuGet restore happens inside `dotnet build`; add `~/.nuget/packages` caching keyed on `Directory.Packages.props` when build times warrant it.
- Turbo remote caching is an optional add-on; never cache `bin/`/`obj/` or task outputs that are processes/databases ([workspace-turbo](../../../rules/monorepo/workspace-turbo.md)).
- Branch protection: `verify` is the required status check; deployment pipelines (migration bundles, image builds) are separate workflows outside this scaffold.
