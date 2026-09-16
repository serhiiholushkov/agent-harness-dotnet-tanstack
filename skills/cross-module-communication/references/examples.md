# Cross-module examples

Reference vocabulary: `Orders` consumes `Catalog`. Substitute concepts; keep shapes.

## Batch contract (owning module)

```csharp
// Catalog.Contracts/ICatalogApi.cs
public interface ICatalogApi
{
    Task<ProductSummary?> GetAsync(Guid productId, CancellationToken cancellationToken = default);

    Task<IReadOnlyDictionary<Guid, ProductSummary>> GetManyAsync(
        IReadOnlyCollection<Guid> productIds,
        CancellationToken cancellationToken = default);
}
```

`CatalogApi` in `PublicApi/` forwards `GetManyAsync` to a `GetProductsByIds` slice performing one `WHERE Id IN (...)` query.

## Page composition in the consuming slice

```csharp
// Orders/Features/Orders/GetOrders.cs — handler excerpt
internal sealed record Response(Guid Id, IReadOnlyList<LineResponse> Lines);
internal sealed record LineResponse(Guid ProductId, string? ProductName, int Quantity);

internal sealed class Handler(OrdersDbContext context, ICatalogApi catalogApi, IUserContext userContext)
    : IQueryHandler<Query, IReadOnlyList<Response>>
{
    public async Task<Result<IReadOnlyList<Response>>> Handle(Query query, CancellationToken cancellationToken)
    {
        List<Order> orders = await context.Orders
            .AsNoTracking()
            .Where(o => o.CustomerId == userContext.UserId)
            .OrderByDescending(o => o.PlacedAt)
            .Take(50)
            .ToListAsync(cancellationToken);

        Guid[] productIds = orders
            .SelectMany(o => o.Lines.Select(l => l.ProductId))
            .Distinct()
            .ToArray();

        IReadOnlyDictionary<Guid, ProductSummary> products =
            await catalogApi.GetManyAsync(productIds, cancellationToken);

        List<Response> responses = orders.Select(o => new Response(
            o.Id,
            o.Lines.Select(l => new LineResponse(
                l.ProductId,
                products.GetValueOrDefault(l.ProductId)?.Name,
                l.Quantity)).ToList()))
            .ToList();

        return responses;
    }
}
```

One local query + one batch call per page; missing products render as `null` names rather than failing the list.

## Snapshot at write time

```csharp
// Orders/Features/Orders/PlaceOrder.cs — handler excerpt
IReadOnlyDictionary<Guid, ProductSummary> products =
    await catalogApi.GetManyAsync(requestedIds, cancellationToken);

foreach (LineRequest line in command.Lines)
{
    if (!products.TryGetValue(line.ProductId, out ProductSummary? product))
    {
        return Result.Failure<Guid>(OrderErrors.ProductUnavailable(line.ProductId));
    }

    order.AddLine(line.ProductId, product.Name, product.PriceCents, line.Quantity);
    // name + price are copied into the order line — the historical record;
    // later catalog changes must not alter a placed order
}
```

## Inbox consumer in the reacting module

```csharp
// Orders/Features/Orders/OnProductPriceChanged.cs
namespace Orders.Features.Orders;

internal sealed class OnProductPriceChanged(OrdersDbContext context)
    : IIntegrationEventHandler<ProductPriceChangedIntegrationEvent>
{
    public async Task Handle(ProductPriceChangedIntegrationEvent @event, CancellationToken cancellationToken)
    {
        // idempotent by inbox (keyed by event id); writes only the orders schema
        await context.ProductPriceProjections
            .Where(p => p.ProductId == @event.ProductId)
            .ExecuteUpdateAsync(
                set => set.SetProperty(p => p.PriceCents, @event.PriceCents),
                cancellationToken);
    }
}
```

The projection table `orders.product_price_projections` exists solely so Orders can sort/filter by current price without touching the `catalog` schema. Placed order lines keep their snapshot.

## Consumer-side test double

```csharp
var catalogApi = Substitute.For<ICatalogApi>();
catalogApi.GetManyAsync(Arg.Any<IReadOnlyCollection<Guid>>(), Arg.Any<CancellationToken>())
    .Returns(new Dictionary<Guid, ProductSummary>
    {
        [productId] = new(productId, "Widget", 1999),
    });
```

If faking the contract takes more than a few lines, the contract is too big — a design signal to raise, not to work around.
