# Module scaffold reference

Concrete file contents for a new module `Shipping`. Substitute the module name; keep the shapes.

## Project files

`apps/api/src/Modules/Shipping/Shipping.Contracts/Shipping.Contracts.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <ItemGroup>
    <ProjectReference Include="..\..\..\Common\Common.SharedKernel\Common.SharedKernel.csproj" />
  </ItemGroup>
</Project>
```

`apps/api/src/Modules/Shipping/Shipping/Shipping.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <ItemGroup>
    <ProjectReference Include="..\Shipping.Contracts\Shipping.Contracts.csproj" />
    <ProjectReference Include="..\..\..\Common\Common.Application\Common.Application.csproj" />
    <ProjectReference Include="..\..\..\Common\Common.Infrastructure\Common.Infrastructure.csproj" />
    <ProjectReference Include="..\..\..\Common\Common.Presentation\Common.Presentation.csproj" />
    <!-- Contracts of modules this module consumes, e.g.: -->
    <ProjectReference Include="..\..\Catalog\Catalog.Contracts\Catalog.Contracts.csproj" />
  </ItemGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.EntityFrameworkCore" />
    <PackageReference Include="Npgsql.EntityFrameworkCore.PostgreSQL" />
    <PackageReference Include="EFCore.NamingConventions" />
    <PackageReference Include="FluentValidation.DependencyInjectionExtensions" />
    <PackageReference Include="Scrutor" />
  </ItemGroup>
  <ItemGroup>
    <InternalsVisibleTo Include="Shipping.UnitTests" />
    <InternalsVisibleTo Include="IntegrationTests" />
  </ItemGroup>
</Project>
```

Versions come from `Directory.Packages.props` — no `Version` attributes here. Target framework, nullable, analyzers come from `Directory.Build.props`.

## Contracts

```csharp
namespace Shipping.Contracts;

public interface IShippingApi
{
    Task<ShipmentSummary?> GetAsync(Guid shipmentId, CancellationToken cancellationToken = default);
}

public sealed record ShipmentSummary(Guid Id, Guid OrderId, string Status);
```

## DbContext

```csharp
namespace Shipping.Database;

internal sealed class ShippingDbContext(
    DbContextOptions<ShippingDbContext> options,
    IDomainEventsDispatcher domainEventsDispatcher) : DbContext(options)
{
    internal DbSet<Shipment> Shipments { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.HasDefaultSchema(Schemas.Shipping);
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(ShippingDbContext).Assembly);
    }
}
```

Plus `Schemas.Shipping = "shipping"` beside the other schema constants, and one `IEntityTypeConfiguration<Shipment>` per entity under `Database/Configurations/`.

## Module entry

```csharp
namespace Shipping;

public static class ShippingModule
{
    public static IServiceCollection AddShippingModule(this IServiceCollection services, IConfiguration configuration)
    {
        services.AddDbContext<ShippingDbContext>(options => options
            .UseNpgsql(
                configuration.GetConnectionString("Database"),
                npgsql => npgsql
                    .MigrationsHistoryTable(HistoryRepository.DefaultTableName, Schemas.Shipping)
                    .MigrationsAssembly(typeof(ShippingDbContext).Assembly.FullName))
            .UseSnakeCaseNamingConvention());

        Assembly assembly = typeof(ShippingModule).Assembly;

        services.Scan(scan => scan.FromAssemblies(assembly)
            .AddClasses(c => c.AssignableTo(typeof(ICommandHandler<>)), publicOnly: false)
                .AsImplementedInterfaces().WithScopedLifetime()
            .AddClasses(c => c.AssignableTo(typeof(ICommandHandler<,>)), publicOnly: false)
                .AsImplementedInterfaces().WithScopedLifetime()
            .AddClasses(c => c.AssignableTo(typeof(IQueryHandler<,>)), publicOnly: false)
                .AsImplementedInterfaces().WithScopedLifetime());

        services.Decorate(typeof(ICommandHandler<,>), typeof(ValidationDecorator.CommandHandler<,>));
        services.Decorate(typeof(ICommandHandler<,>), typeof(LoggingDecorator.CommandHandler<,>));

        services.AddValidatorsFromAssembly(assembly, includeInternalTypes: true);
        services.AddEndpoints(assembly);

        services.AddScoped<IShippingApi, ShippingApi>();

        return services;
    }

    public static IEndpointRouteBuilder MapShippingEndpoints(this IEndpointRouteBuilder app)
    {
        RouteGroupBuilder v1 = app.MapGroup("api/v1/shipping")
            .WithTags(Tags.Shipping)
            .WithGroupName(ApiVersions.V1);

        v1.MapEndpoints(ApiVersions.V1);

        return app;
    }
}
```

## Architecture test additions

```csharp
[Fact]
public void Shipping_Should_NotReference_SiblingImplementations() =>
    Types.InAssembly(ShippingAssembly)
        .ShouldNot()
        .HaveDependencyOnAny("Catalog.Features", "Catalog.Database", "Orders.Features", "Orders.Database")
        .GetResult().IsSuccessful.ShouldBeTrue();

[Fact]
public void Siblings_Should_NotReference_ShippingImplementation() =>
    Types.InAssemblies(new[] { CatalogAssembly, OrdersAssembly })
        .ShouldNot()
        .HaveDependencyOnAny("Shipping.Features", "Shipping.Database", "Shipping.PublicApi")
        .GetResult().IsSuccessful.ShouldBeTrue();

[Fact]
public void ShippingContracts_Should_BePure() =>
    Types.InAssembly(typeof(IShippingApi).Assembly)
        .ShouldNot()
        .HaveDependencyOnAny("Shipping", "Microsoft.EntityFrameworkCore")
        .GetResult().IsSuccessful.ShouldBeTrue();
```

## Test projects

Mirror the existing pattern: `apps/api/tests/Modules/Shipping/Shipping.UnitTests/` (xUnit + Shouldly + NSubstitute, referencing the implementation project via `InternalsVisibleTo`), and extend `IntegrationTests` with the module's HTTP-surface tests. Cover the module's decorator pipeline explicitly — this module's `Decorate` calls are its own.
