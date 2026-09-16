# API solution

Layout under `apps/api/`:

```text
apps/api/
├── package.json
├── Api.slnx
├── Directory.Build.props
├── Directory.Packages.props
├── openapi/                    # build output, committed
├── src/
│   ├── Common/
│   │   ├── Common.SharedKernel/
│   │   ├── Common.Application/
│   │   ├── Common.Infrastructure/
│   │   └── Common.Presentation/
│   ├── Modules/                # empty until add-module
│   └── Api/
│       ├── Api.csproj
│       ├── Program.cs
│       ├── Properties/launchSettings.json
│       └── appsettings.json
└── tests/
    ├── ArchitectureTests/
    └── IntegrationTests/
```

## package.json (thin wrapper — no JS, no build logic)

```json
{
  "name": "@repo/api",
  "private": true,
  "scripts": {
    "dev": "dotnet watch run --project src/Api",
    "build": "dotnet build Api.slnx -c Release",
    "test": "dotnet test Api.slnx -c Release --no-build",
    "lint": "dotnet format Api.slnx --verify-no-changes"
  }
}
```

## Api.slnx

```xml
<Solution>
  <Folder Name="/src/">
    <Project Path="src/Common/Common.SharedKernel/Common.SharedKernel.csproj" />
    <Project Path="src/Common/Common.Application/Common.Application.csproj" />
    <Project Path="src/Common/Common.Infrastructure/Common.Infrastructure.csproj" />
    <Project Path="src/Common/Common.Presentation/Common.Presentation.csproj" />
    <Project Path="src/Api/Api.csproj" />
  </Folder>
  <Folder Name="/tests/">
    <Project Path="tests/ArchitectureTests/ArchitectureTests.csproj" />
    <Project Path="tests/IntegrationTests/IntegrationTests.csproj" />
  </Folder>
</Solution>
```

Module projects are added here by [add-module](../../add-module/SKILL.md).

## Directory.Build.props

```xml
<Project>
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <AnalysisLevel>latest</AnalysisLevel>
    <AnalysisMode>All</AnalysisMode>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <CodeAnalysisTreatWarningsAsErrors>true</CodeAnalysisTreatWarningsAsErrors>
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
  </PropertyGroup>
</Project>
```

## Directory.Packages.props

Versions are pinned examples — refresh patches at scaffold time, keep the majors.

```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>
  <ItemGroup>
    <!-- Application -->
    <PackageVersion Include="FluentValidation.DependencyInjectionExtensions" Version="12.1.1" />
    <PackageVersion Include="Scrutor" Version="7.0.0" />
    <PackageVersion Include="Microsoft.EntityFrameworkCore" Version="10.0.0" />
    <PackageVersion Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="10.0.0" />
    <PackageVersion Include="EFCore.NamingConventions" Version="10.0.0" />
    <PackageVersion Include="Microsoft.Extensions.Caching.Hybrid" Version="10.0.0" />
    <!-- Presentation / OpenAPI -->
    <PackageVersion Include="Microsoft.AspNetCore.OpenApi" Version="10.0.0" />
    <PackageVersion Include="Microsoft.Extensions.ApiDescription.Server" Version="10.0.0" />
    <PackageVersion Include="Scalar.AspNetCore" Version="2.16.0" />
    <!-- Infrastructure -->
    <PackageVersion Include="Microsoft.AspNetCore.Authentication.JwtBearer" Version="10.0.0" />
    <PackageVersion Include="Microsoft.Extensions.Diagnostics.HealthChecks" Version="10.0.0" />
    <PackageVersion Include="AspNetCore.HealthChecks.NpgSql" Version="9.0.0" />
    <PackageVersion Include="OpenTelemetry.Extensions.Hosting" Version="1.16.0" />
    <PackageVersion Include="OpenTelemetry.Instrumentation.AspNetCore" Version="1.16.0" />
    <PackageVersion Include="OpenTelemetry.Instrumentation.Http" Version="1.16.0" />
    <PackageVersion Include="Npgsql.OpenTelemetry" Version="10.0.0" />
    <PackageVersion Include="OpenTelemetry.Exporter.OpenTelemetryProtocol" Version="1.16.0" />
    <!-- Tests -->
    <PackageVersion Include="xunit" Version="2.9.3" />
    <PackageVersion Include="xunit.runner.visualstudio" Version="3.1.5" />
    <PackageVersion Include="Microsoft.NET.Test.Sdk" Version="18.7.0" />
    <PackageVersion Include="Shouldly" Version="4.3.0" />
    <PackageVersion Include="NSubstitute" Version="5.3.0" />
    <PackageVersion Include="NetArchTest.Rules" Version="1.3.2" />
    <PackageVersion Include="Testcontainers.PostgreSql" Version="4.12.0" />
    <PackageVersion Include="Microsoft.AspNetCore.Mvc.Testing" Version="10.0.0" />
    <PackageVersion Include="coverlet.collector" Version="10.0.1" />
  </ItemGroup>
</Project>
```

## Project references

- `Common.SharedKernel` — no references.
- `Common.Application` → `Common.SharedKernel`; packages: FluentValidation, Microsoft.Extensions.Logging.Abstractions.
- `Common.Infrastructure` → `Common.Application`; packages: EF Core, Npgsql, NamingConventions, Hybrid cache, JwtBearer, health checks, OpenTelemetry.
- `Common.Presentation` → `Common.Application`; packages: Microsoft.AspNetCore.OpenApi (FrameworkReference `Microsoft.AspNetCore.App`).
- `Api` → all `Common.*` (modules added later); packages: Microsoft.AspNetCore.OpenApi, Microsoft.Extensions.ApiDescription.Server (PrivateAssets=all), Scalar.AspNetCore.

## Api.csproj — build-time OpenAPI

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <OpenApiDocumentsDirectory>$(MSBuildProjectDirectory)/../../openapi</OpenApiDocumentsDirectory>
    <OpenApiGenerateDocuments>true</OpenApiGenerateDocuments>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.OpenApi" />
    <PackageReference Include="Scalar.AspNetCore" />
    <PackageReference Include="Microsoft.Extensions.ApiDescription.Server">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
    </PackageReference>
    <ProjectReference Include="..\Common\Common.SharedKernel\Common.SharedKernel.csproj" />
    <ProjectReference Include="..\Common\Common.Application\Common.Application.csproj" />
    <ProjectReference Include="..\Common\Common.Infrastructure\Common.Infrastructure.csproj" />
    <ProjectReference Include="..\Common\Common.Presentation\Common.Presentation.csproj" />
  </ItemGroup>
</Project>
```

Build-time generation invokes the app's `Program` per document, so keep `Program.cs` side-effect-free at startup (no migrations, no external calls) — health checks and readiness gates handle runtime concerns.

## Properties/launchSettings.json (API on 5000)

```json
{
  "profiles": {
    "http": {
      "commandName": "Project",
      "applicationUrl": "http://localhost:5000",
      "environmentVariables": { "ASPNETCORE_ENVIRONMENT": "Development" }
    }
  }
}
```

## Program.cs (skeleton, no modules yet)

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddCommonInfrastructure(builder.Configuration);
builder.Services.AddCommonPresentation();          // ProblemDetails, exception handler, OpenAPI options
builder.Services.AddOpenApiDocuments();            // AddOpenApi(ApiVersions.V1) per active version
// Modules (dependency order) — added by add-module:
// builder.Services.AddCatalogModule(builder.Configuration);

WebApplication app = builder.Build();

app.UseExceptionHandler();
app.UseAuthentication();
app.UseAuthorization();

// app.MapCatalogEndpoints();                      // added by add-module

app.MapOpenApi();

if (app.Environment.IsDevelopment())
{
    app.MapScalarApiReference();                   // /docs — dev only
}

app.MapHealthChecks("/health/live", new HealthCheckOptions { Predicate = _ => false })
   .ExcludeFromDescription();
app.MapHealthChecks("/health/ready", new HealthCheckOptions
   {
       Predicate = check => check.Tags.Contains("ready")
   })
   .ExcludeFromDescription();

await app.RunAsync();

public partial class Program;                      // integration-test host handle
```

Kernel types referenced here: [common-kernel.md](common-kernel.md).

## Test projects

- `ArchitectureTests` — xUnit + NetArchTest + Shouldly; seeded with convention tests (handlers/endpoints sealed, module types internal) that enumerate module assemblies as they appear.
- `IntegrationTests` — xUnit + Testcontainers.PostgreSql + Mvc.Testing; `IntegrationTestWebAppFactory` overriding `ConnectionStrings:Database` with the container's string and running migrations per module context; plus the OpenAPI boot test (documents generate, operation ids unique, public routes present exactly once).
