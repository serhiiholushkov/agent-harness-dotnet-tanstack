---
title: Configuration
description: Typed options validated at startup; no raw configuration reads; environments differ by values.
appliesTo: 'apps/api/src/**/*.cs'
---

# Configuration

- Every setting binds to a typed options class registered with `AddOptions<T>().BindConfiguration(T.SectionName).ValidateDataAnnotations().ValidateOnStart()`. A missing or malformed setting fails the boot with a named error — never a `NullReferenceException` mid-request.
- Options classes are named `{Name}Options`, expose a `public const string SectionName`, and carry data annotations for their invariants.
- No `IConfiguration["…"]` or `configuration.GetSection(…)` reads inside slices, handlers, or module services. Slices inject `IOptions<T>`/`IOptionsSnapshot<T>`.
- No `configuration.GetConnectionString("…")!` — the module registration reads the connection string once, and its absence fails startup.
- Environment differences are values, not code paths: add `Shutdown:DrainSeconds` or `RateLimit:PermitLimit` instead of branching on `IHostEnvironment`. Environment checks are allowed only in `Program.cs`, for developer-only middleware such as the Scalar reference UI.
- Secrets arrive via environment variables or a secret store; local development uses user secrets. Committed `appsettings*.json` contain no secrets ([../common/security.md](../common/security.md)).
- New configuration ships with its validation and a test that boot fails when it is missing.

Caught by: `ValidateOnStart` at boot, integration tests booting the real host, review for raw reads and env branching.
