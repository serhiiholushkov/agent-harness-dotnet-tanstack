# Root files

Pinned versions below are **examples current at authoring time** — pin to the latest current patch of the same majors when scaffolding.

## package.json (root)

```json
{
  "name": "repo",
  "private": true,
  "packageManager": "pnpm@10.17.0",
  "engines": { "node": ">=24" },
  "scripts": {
    "dev": "turbo run dev",
    "lint": "turbo run lint",
    "typecheck": "turbo run typecheck",
    "test": "turbo run test",
    "build": "turbo run build",
    "generate": "turbo run generate",
    "check:drift": "turbo run build generate && git diff --exit-code -- apps/api/openapi packages/api-client/src",
    "verify": "turbo run lint typecheck test build && pnpm run check:drift"
  },
  "devDependencies": {
    "turbo": "^2.5.0"
  }
}
```

## pnpm-workspace.yaml

```yaml
packages:
  - apps/*
  - packages/*
```

## turbo.json

```json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": ["global.json", ".config/dotnet-tools.json"],
  "tasks": {
    "@repo/api#build": {
      "cache": false,
      "outputs": ["openapi/*.json"]
    },
    "@repo/api#test": { "dependsOn": ["build"], "cache": false },
    "@repo/api-client#generate": {
      "dependsOn": ["@repo/api#build"],
      "inputs": ["$TURBO_ROOT$/apps/api/openapi/*.json"],
      "outputs": ["src/*.gen.ts"]
    },
    "@repo/web#build": {
      "dependsOn": ["^build", "@repo/api-client#generate"],
      "outputs": [".output/**", "dist/**"]
    },
    "typecheck": { "dependsOn": ["^build", "@repo/api-client#generate"] },
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**", ".output/**"] },
    "lint": {},
    "test": {},
    "dev": { "cache": false, "persistent": true }
  }
}
```

Root `package#task` overrides replace base config — if you add fields to a base task later, re-check the overrides.

## global.json

```json
{
  "sdk": {
    "version": "10.0.100",
    "rollForward": "latestPatch"
  }
}
```

## .config/dotnet-tools.json

```json
{
  "version": 1,
  "isRoot": true,
  "tools": {
    "dotnet-ef": {
      "version": "10.0.0",
      "commands": ["dotnet-ef"]
    }
  }
}
```

## .gitignore (essentials)

```gitignore
node_modules/
.turbo/
dist/
.output/
.nitro/
.tanstack/
bin/
obj/
*.user
.env
.env.*
!.env.example
coverage/
TestResults/
```

Generated-but-committed files must **not** be ignored: `apps/api/openapi/*.json`, `packages/api-client/src/*.gen.ts`, `apps/web/src/routeTree.gen.ts`, EF migrations.

## .env.example

```dotenv
# API (user secrets preferred locally)
ConnectionStrings__Database=Host=localhost;Port=5432;Database=app;Username=app;Password=dev-only

# Web server runtime (server-only; never exposed to the browser)
API_BASE_URL=http://localhost:5000
API_TIMEOUT_MS=5000
```

## docker-compose.yml

```yaml
services:
  db:
    image: postgres:17
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: app
      POSTGRES_PASSWORD: dev-only
    ports:
      - '5432:5432'
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U app -d app']
      interval: 5s
      timeout: 3s
      retries: 10

volumes:
  db-data:
```
