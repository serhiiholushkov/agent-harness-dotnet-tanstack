# Slice examples

Full reference implementations for the two slice shapes. Vocabulary (`Catalog`, `Product`) is the reference example set — substitute your module's concepts.

## Query slice with caching

```csharp
namespace Catalog.Features.Products;

internal static class GetProduct
{
    internal sealed record Query(Guid ProductId) : IQuery<Response>;

    internal sealed record Response(Guid Id, string Name, int PriceCents, bool IsArchived);

    internal sealed class Handler(CatalogDbContext context, HybridCache cache) : IQueryHandler<Query, Response>
    {
        public async Task<Result<Response>> Handle(Query query, CancellationToken cancellationToken)
        {
            Response? product = await cache.GetOrCreateAsync(
                ProductCacheKeys.ById(query.ProductId),
                async cancellation => await context.Products
                    .Where(p => p.Id == query.ProductId)
                    .Select(p => new Response(p.Id, p.Name, p.PriceCents, p.IsArchived))
                    .SingleOrDefaultAsync(cancellation),
                cancellationToken: cancellationToken);

            return product is null
                ? Result.Failure<Response>(ProductErrors.NotFound(query.ProductId))
                : product;
        }
    }

    internal sealed class Endpoint : IEndpoint
    {
        public string Version => ApiVersions.V1;

        public void MapEndpoint(IEndpointRouteBuilder app) =>
            app.MapGet("products/{id:guid}", async (
                Guid id, IQueryHandler<Query, Response> handler, CancellationToken cancellationToken) =>
            {
                Result<Response> result = await handler.Handle(new Query(id), cancellationToken);

                return result.Match(TypedResults.Ok, CustomResults.Problem);
            })
            .WithName("getCatalogProductV1")
            .WithSummary("Get a catalog product")
            .Produces<Response>(StatusCodes.Status200OK)
            .ProducesProblem(StatusCodes.Status401Unauthorized)
            .ProducesProblem(StatusCodes.Status404NotFound)
            .RequireAuthorization();
    }
}
```

Notes:

- Projection (`Select`) goes straight to `Response`; no tracked entity, no `AsNoTracking` needed.
- The cache key class is the module's single key registry; the write slices invalidate it.
- Queries usually skip the `Validator` — a malformed `Guid` never binds; in-handler checks cover the rest.

## Command slice with a wire request, creation, and event

```csharp
namespace Catalog.Features.Products;

internal static class CreateProduct
{
    internal sealed record Command(string Name, int PriceCents) : ICommand<Guid>;

    internal sealed class Validator : AbstractValidator<Command>
    {
        public Validator()
        {
            RuleFor(c => c.Name).NotEmpty().MaximumLength(200);
            RuleFor(c => c.PriceCents).GreaterThanOrEqualTo(0);
        }
    }

    internal sealed class Handler(
        CatalogDbContext context,
        IDateTimeProvider dateTimeProvider) : ICommandHandler<Command, Guid>
    {
        public async Task<Result<Guid>> Handle(Command command, CancellationToken cancellationToken)
        {
            bool nameTaken = await context.Products
                .AnyAsync(p => p.Name == command.Name, cancellationToken);

            if (nameTaken)
            {
                return Result.Failure<Guid>(ProductErrors.NameTaken(command.Name));
            }

            var product = Product.Create(command.Name, command.PriceCents, dateTimeProvider.UtcNow);
            // Create assigns Guid.NewGuid() first, then raises ProductCreatedDomainEvent

            context.Products.Add(product);
            await context.SaveChangesAsync(cancellationToken);

            return product.Id;
        }
    }

    internal sealed class Endpoint : IEndpoint
    {
        internal sealed record Request(string Name, int PriceCents);

        public string Version => ApiVersions.V1;

        public void MapEndpoint(IEndpointRouteBuilder app) =>
            app.MapPost("products", async (
                Request request, ICommandHandler<Command, Guid> handler, CancellationToken cancellationToken) =>
            {
                var command = new Command(request.Name, request.PriceCents);

                Result<Guid> result = await handler.Handle(command, cancellationToken);

                return result.Match(
                    id => TypedResults.Created($"/api/v1/catalog/products/{id}", id),
                    CustomResults.Problem);
            })
            .WithName("createCatalogProductV1")
            .WithSummary("Create a catalog product")
            .Produces<Guid>(StatusCodes.Status201Created)
            .ProducesProblem(StatusCodes.Status400BadRequest)
            .ProducesProblem(StatusCodes.Status401Unauthorized)
            .ProducesProblem(StatusCodes.Status409Conflict)
            .RequireAuthorization();
    }
}
```

Notes:

- A separate `Request` record is warranted when the wire shape and the command diverge (or will); collapse them knowingly otherwise.
- The identifier is assigned inside `Product.Create` **before** the domain event is raised — a database-generated key would put `Guid.Empty` in the event.
- Money is integer minor units (`PriceCents`) per the type-mapping table.
- For principal-owned entities, the ownership filter belongs in every query predicate: `.Where(o => o.Id == id && o.CustomerId == userContext.UserId)` — same 404 for missing and foreign rows.

## Test skeletons

```csharp
public sealed class CreateProductValidatorTests
{
    private readonly CreateProduct.Validator _validator = new();

    [Fact]
    public void Should_Fail_When_NameEmpty() =>
        _validator.Validate(new CreateProduct.Command("", 100))
            .IsValid.ShouldBeFalse();
}

public sealed class CreateProductHandlerTests
{
    [Fact]
    public async Task Should_ReturnConflict_When_NameTaken()
    {
        // arrange context with an existing "Widget", construct the nested handler directly
        Result<Guid> result = await handler.Handle(new CreateProduct.Command("Widget", 100), CancellationToken.None);

        result.IsFailure.ShouldBeTrue();
        result.Error.Type.ShouldBe(ErrorType.Conflict);
    }
}
```

Handler tests bypass the decorators — keep the module's decorator pipeline covered separately ([api-testing](../../../rules/testing/api-testing.md)).
