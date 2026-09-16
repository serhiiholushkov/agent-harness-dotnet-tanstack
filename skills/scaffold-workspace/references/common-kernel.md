# Common.\* kernel starter

The minimal application-neutral primitives the skeleton compiles with. Everything here is stable, cross-cutting, and module-free — the bar for `Common.*`.

## Common.SharedKernel

```csharp
namespace Common.SharedKernel;

public enum ErrorType
{
    Failure,
    Validation,
    Problem,
    NotFound,
    Conflict,
    Unauthorized,
    Forbidden,
}

public record Error(string Code, string Description, ErrorType Type)
{
    public static readonly Error None = new(string.Empty, string.Empty, ErrorType.Failure);

    public static Error Failure(string code, string description) => new(code, description, ErrorType.Failure);
    public static Error Validation(string code, string description) => new(code, description, ErrorType.Validation);
    public static Error Problem(string code, string description) => new(code, description, ErrorType.Problem);
    public static Error NotFound(string code, string description) => new(code, description, ErrorType.NotFound);
    public static Error Conflict(string code, string description) => new(code, description, ErrorType.Conflict);
    public static Error Unauthorized(string code, string description) => new(code, description, ErrorType.Unauthorized);
    public static Error Forbidden(string code, string description) => new(code, description, ErrorType.Forbidden);
}

public class Result
{
    protected Result(bool isSuccess, Error error)
    {
        if (isSuccess && error != Error.None || !isSuccess && error == Error.None)
        {
            throw new ArgumentException("Invalid error state", nameof(error));
        }

        IsSuccess = isSuccess;
        Error = error;
    }

    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;
    public Error Error { get; }

    public static Result Success() => new(true, Error.None);
    public static Result Failure(Error error) => new(false, error);
    public static Result<T> Success<T>(T value) => new(value, true, Error.None);
    public static Result<T> Failure<T>(Error error) => new(default, false, error);
}

public sealed class Result<T> : Result
{
    private readonly T? _value;

    internal Result(T? value, bool isSuccess, Error error) : base(isSuccess, error) => _value = value;

    public T Value => IsSuccess
        ? _value!
        : throw new InvalidOperationException("The value of a failure result cannot be accessed.");

    public static implicit operator Result<T>(T value) => Success(value);
}

public static class ResultExtensions
{
    public static TOut Match<TOut>(this Result result, Func<TOut> onSuccess, Func<Result, TOut> onFailure) =>
        result.IsSuccess ? onSuccess() : onFailure(result);

    public static TOut Match<TIn, TOut>(this Result<TIn> result, Func<TIn, TOut> onSuccess, Func<Result<TIn>, TOut> onFailure) =>
        result.IsSuccess ? onSuccess(result.Value) : onFailure(result);
}

public interface IDateTimeProvider
{
    DateTime UtcNow { get; }
}

public abstract class Entity
{
    private readonly List<IDomainEvent> _domainEvents = [];

    public IReadOnlyCollection<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    public void Raise(IDomainEvent domainEvent) => _domainEvents.Add(domainEvent);
    public void ClearDomainEvents() => _domainEvents.Clear();
}

public interface IDomainEvent;
```

## Common.Application

```csharp
namespace Common.Application.Messaging;

public interface ICommand;
public interface ICommand<TResponse>;
public interface IQuery<TResponse>;

public interface ICommandHandler<in TCommand> where TCommand : ICommand
{
    Task<Result> Handle(TCommand command, CancellationToken cancellationToken);
}

public interface ICommandHandler<in TCommand, TResponse> where TCommand : ICommand<TResponse>
{
    Task<Result<TResponse>> Handle(TCommand command, CancellationToken cancellationToken);
}

public interface IQueryHandler<in TQuery, TResponse> where TQuery : IQuery<TResponse>
{
    Task<Result<TResponse>> Handle(TQuery query, CancellationToken cancellationToken);
}
```

Plus, in the same project: `ValidationDecorator` (runs registered `IValidator<TCommand>`s, short-circuits into a `Validation` error), `LoggingDecorator` (structured start/finish/failure), and a `ValidationError` aggregating failures. Decorators close over the inner handler interface so Scrutor's `Decorate` wires them per module.

## Common.Presentation

```csharp
namespace Common.Presentation.Endpoints;

public interface IEndpoint
{
    string Version { get; }
    void MapEndpoint(IEndpointRouteBuilder app);
}

public static class EndpointExtensions
{
    public static IServiceCollection AddEndpoints(this IServiceCollection services, Assembly assembly)
    {
        ServiceDescriptor[] descriptors = assembly.DefinedTypes
            .Where(t => t is { IsAbstract: false, IsInterface: false } && t.IsAssignableTo(typeof(IEndpoint)))
            .Select(t => ServiceDescriptor.Transient(typeof(IEndpoint), t))
            .ToArray();

        services.TryAddEnumerable(descriptors);

        return services;
    }

    public static IEndpointRouteBuilder MapEndpoints(this RouteGroupBuilder group, string version)
    {
        IEnumerable<IEndpoint> endpoints = group.ServiceProvider
            .GetRequiredService<IEnumerable<IEndpoint>>()
            .Where(e => e.Version == version);

        foreach (IEndpoint endpoint in endpoints)
        {
            endpoint.MapEndpoint(group);
        }

        return group;
    }
}

public static class ApiVersions
{
    public const string V1 = "v1";
}

public static class Tags
{
    // one constant per module, added by add-module
}
```

`CustomResults`:

```csharp
namespace Common.Presentation;

public static class CustomResults
{
    public static ProblemHttpResult Problem(Result result)
    {
        Error error = result.Error;

        return TypedResults.Problem(
            title: GetTitle(error),
            detail: GetDetail(error),
            type: GetType(error.Type),
            statusCode: error.Type switch
            {
                ErrorType.Validation or ErrorType.Problem => StatusCodes.Status400BadRequest,
                ErrorType.Unauthorized => StatusCodes.Status401Unauthorized,
                ErrorType.Forbidden => StatusCodes.Status403Forbidden,
                ErrorType.NotFound => StatusCodes.Status404NotFound,
                ErrorType.Conflict => StatusCodes.Status409Conflict,
                _ => StatusCodes.Status500InternalServerError,
            });
    }
    // GetTitle/GetDetail return generic text for ErrorType.Failure (5xx) and the
    // error's code/description otherwise; validation errors attach field failures.
}
```

OpenAPI options (per document) in `AddOpenApiDocuments()`:

```csharp
services.AddOpenApi(ApiVersions.V1, options =>
{
    options.CreateSchemaReferenceId = type =>
        type.Type.DeclaringType is { } slice
            ? $"{slice.Name}{type.Type.Name}"      // GetProduct.Response -> GetProductResponse
            : OpenApiOptions.CreateDefaultSchemaReferenceId(type);
});
```

Serialization decided once (in the host's `ConfigureHttpJsonOptions`): `JsonStringEnumConverter`, camelCase (default), one global ignore condition.

`GlobalExceptionHandler : IExceptionHandler` — logs with structured fields, returns the shared `ProblemDetails` shape, generic message for 5xx. Registered with `AddProblemDetails()` in `AddCommonPresentation()`.

## Common.Infrastructure

Starter content: `AddCommonInfrastructure(IConfiguration)` registering `IDateTimeProvider`, `HybridCache` (`AddHybridCache()`), authentication (JWT bearer) + `IUserContext`, health checks (`AddNpgSql` tagged `"ready"` once a connection string exists), OpenTelemetry (ASP.NET Core, HttpClient, Npgsql), and the security-header middleware (`nosniff`, HSTS). The outbox/inbox infrastructure (`IntegrationEvent` base, `IIntegrationEventHandler<T>`, `IDomainEventsDispatcher`, SaveChanges interceptor, processor) also lives here and is consumed by modules; keep it event-transport-neutral.
