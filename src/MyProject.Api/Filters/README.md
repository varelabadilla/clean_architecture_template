# Api / Filters

## Why This Folder Exists

Filters run **at specific points in the MVC action pipeline** — before or after model binding, before or after action execution, before or after result execution. Unlike middleware (which runs for every HTTP request), filters can be scoped to specific controllers or actions.

Common uses: idempotency key enforcement, specific authorization checks, response caching headers, and action-level logging.

---

## What Belongs Here

- `IActionFilter` implementations — run around action execution
- `IAuthorizationFilter` / `IAsyncAuthorizationFilter` — action-level authorization
- `IResultFilter` — run around result execution
- `IExceptionFilter` — action-scoped exception handling (prefer `GlobalExceptionMiddleware` for global handling)
- `IIdempotencyFilter` — prevents duplicate POST processing

---

## What Does NOT Belong Here

- Global exception handling (use `Middleware/GlobalExceptionMiddleware`)
- Business logic
- Infrastructure concerns

---

## Recommended Naming Conventions

| Type | Convention | Example |
| --- | --- | --- |
| Filter class | Descriptive + `Filter` | `IdempotencyFilter`, `RateLimitFilter` |
| Attribute-based filter | Descriptive + `Attribute` | `RequiresPermissionAttribute` |

---

## Example: IdempotencyFilter

```csharp
// Prevents duplicate processing of POST requests with the same Idempotency-Key header
public class IdempotencyFilter : IAsyncActionFilter
{
    private readonly ICacheService _cache;

    public IdempotencyFilter(ICacheService cache) => _cache = cache;

    public async Task OnActionExecutionAsync(
        ActionExecutingContext context,
        ActionExecutionDelegate next)
    {
        if (!context.HttpContext.Request.Headers
            .TryGetValue("Idempotency-Key", out var key))
        {
            await next();
            return;
        }

        var cacheKey = $"idempotency:{key}";
        var cached = await _cache.GetAsync<object>(cacheKey);

        if (cached is not null)
        {
            context.Result = new OkObjectResult(cached);
            return;
        }

        var executedContext = await next();

        if (executedContext.Result is OkObjectResult okResult)
            await _cache.SetAsync(cacheKey, okResult.Value, TimeSpan.FromHours(24));
    }
}
```

---

## Important Notes

- Use middleware for **cross-cutting concerns that apply to every request**. Use filters for **concerns that apply to specific controllers or actions**.
- In Java/Spring, the equivalent of action filters is `HandlerInterceptor` or Spring AOP `@Around` aspects applied to specific controller methods.
