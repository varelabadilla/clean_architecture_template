# Application / Behaviors

## Why This Folder Exists

Behaviors implement **cross-cutting concerns** that apply to many or all handlers, without those handlers needing to know about them. Logging, validation, transaction management, and caching all need to run around handler execution — but a `PlaceOrderHandler` should only care about placing orders, not about starting transactions or logging timing.

In MediatR, behaviors form a **pipeline**: each request passes through a chain of behaviors before reaching the handler, and the response passes back through them on the way out. This is the Decorator pattern applied to the CQRS pipeline.

---

## What Belongs Here

- `LoggingBehavior<TRequest, TResponse>` — structured logging around every request
- `ValidationBehavior<TRequest, TResponse>` — runs FluentValidation before the handler
- `TransactionBehavior<TRequest, TResponse>` — wraps command handlers in a DB transaction
- `CachingBehavior<TRequest, TResponse>` — returns cached results for queries
- `PerformanceBehavior<TRequest, TResponse>` — warns when handlers run slow

---

## What Does NOT Belong Here

- Business logic of any kind
- Application-specific utilities or helpers
- Domain services
- Framework configuration code

---

## Recommended Naming Conventions

| Type | Convention | Example |
| --- | --- | --- |
| Behavior class | Noun + `Behavior` | `ValidationBehavior`, `LoggingBehavior` |
| Generic type params | `TRequest`, `TResponse` | Standard MediatR naming |

---

## Pipeline Order

Register behaviors in this order in `DependencyInjection.cs`. MediatR executes them in registration order:

```plaintext
Incoming:  LoggingBehavior → ValidationBehavior → TransactionBehavior → Handler
Outgoing:  Handler → TransactionBehavior → ValidationBehavior → LoggingBehavior
```

Logging wraps everything so it can capture total execution time including validation. Validation runs before the transaction so we never open a transaction for an invalid request. Transaction wraps the handler for commands only.

---

## Behavior Implementations

### LoggingBehavior

```csharp
public class LoggingBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    private readonly ILogger<LoggingBehavior<TRequest, TResponse>> _logger;

    public LoggingBehavior(ILogger<LoggingBehavior<TRequest, TResponse>> logger)
        => _logger = logger;

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        var requestName = typeof(TRequest).Name;
        _logger.LogInformation("Handling {RequestName}", requestName);

        var sw = Stopwatch.StartNew();
        var response = await next();
        sw.Stop();

        _logger.LogInformation(
            "Handled {RequestName} in {ElapsedMs}ms", requestName, sw.ElapsedMilliseconds);

        return response;
    }
}
```

### ValidationBehavior

```csharp
public class ValidationBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;

    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators)
        => _validators = validators;

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        if (!_validators.Any())
            return await next();

        var context = new ValidationContext<TRequest>(request);
        var failures = _validators
            .Select(v => v.Validate(context))
            .SelectMany(r => r.Errors)
            .Where(f => f is not null)
            .ToList();

        if (failures.Count != 0)
            throw new ValidationException(failures);

        return await next();
    }
}
```

### TransactionBehavior

```csharp
// Only wraps commands (ICommand marker interface) — not queries
public class TransactionBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : ICommand  // Marker interface defined in Application/Common
{
    private readonly IUnitOfWork _unitOfWork;

    public TransactionBehavior(IUnitOfWork unitOfWork)
        => _unitOfWork = unitOfWork;

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        var response = await next();
        await _unitOfWork.SaveChangesAsync(cancellationToken);
        return response;
    }
}
```

### CachingBehavior

```csharp
// Only caches queries that implement ICacheableQuery
public class CachingBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : ICacheableQuery
{
    private readonly ICacheService _cache;

    public CachingBehavior(ICacheService cache) => _cache = cache;

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        var cached = await _cache.GetAsync<TResponse>(request.CacheKey, cancellationToken);
        if (cached is not null) return cached;

        var response = await next();

        await _cache.SetAsync(request.CacheKey, response, request.CacheDuration, cancellationToken);
        return response;
    }
}
```

---

## Marker Interfaces (defined in Application/Common)

Behaviors use marker interfaces to selectively apply to commands or cacheable queries:

```csharp
// Commands carry write intent
public interface ICommand : IRequest<Result> { }
public interface ICommand<TResponse> : IRequest<Result<TResponse>> { }

// Cacheable queries declare their cache key and duration
public interface ICacheableQuery
{
    string CacheKey { get; }
    TimeSpan? CacheDuration { get; }
}
```

---

## Important Notes

- Behaviors run for **every** matching request. Keep them fast. A slow behavior adds latency to every handler.
- Each behavior should do exactly one thing. A combined "logging and validation" behavior is harder to test and maintain.
- The `TransactionBehavior` should only apply to commands, not queries. Use a marker interface or check `TRequest` type to apply selectively.
- In Java/Spring, the equivalent is AOP with `@Around` advice, or custom `HandlerInterceptor` implementations. The principle is identical — wrap behavior around the handler without the handler knowing.
