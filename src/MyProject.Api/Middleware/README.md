# Api / Middleware

## Why This Folder Exists

Middleware components sit in the **HTTP request/response pipeline** and handle cross-cutting HTTP concerns: global exception handling, request logging, correlation IDs, rate limiting, and response compression. They run for every request, before or after controllers.

Centralizing middleware here keeps controllers and filters clean and ensures consistent behavior across the entire API surface.

---

## What Belongs Here

- `GlobalExceptionMiddleware` — catches unhandled exceptions and returns structured error responses
- `RequestLoggingMiddleware` — logs incoming requests and outgoing responses
- `CorrelationIdMiddleware` — propagates or generates request correlation IDs
- `RateLimitingMiddleware` — custom rate limiting if not using built-in ASP.NET middleware

---

## What Does NOT Belong Here

- Business logic
- Authentication/authorization logic (use ASP.NET Core's built-in auth middleware)
- Response caching configuration (handled in `DependencyInjection/`)

---

## Recommended Naming Conventions

| Type | Convention | Example |
| --- | --- | --- |
| Middleware class | Descriptive + `Middleware` | `GlobalExceptionMiddleware` |
| Extension method | `Use` + description | `UseGlobalExceptionHandler` |

---

## GlobalExceptionMiddleware

The most important middleware in this folder. It translates all unhandled exceptions into a consistent, structured HTTP error response.

```csharp
public class GlobalExceptionMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<GlobalExceptionMiddleware> _logger;

    public GlobalExceptionMiddleware(
        RequestDelegate next,
        ILogger<GlobalExceptionMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception exception)
        {
            _logger.LogError(exception,
                "Unhandled exception for {Method} {Path}",
                context.Request.Method,
                context.Request.Path);

            await HandleExceptionAsync(context, exception);
        }
    }

    private static async Task HandleExceptionAsync(
        HttpContext context, Exception exception)
    {
        var (statusCode, errorCode, message, errors) = exception switch
        {
            ValidationException ve => (
                StatusCodes.Status422UnprocessableEntity,
                "validation_error",
                "One or more validation errors occurred.",
                (object?)ve.Errors),

            NotFoundException nfe => (
                StatusCodes.Status404NotFound,
                "not_found",
                nfe.Message,
                (object?)null),

            ForbiddenException fe => (
                StatusCodes.Status403Forbidden,
                "forbidden",
                fe.Message,
                (object?)null),

            ConflictException ce => (
                StatusCodes.Status409Conflict,
                "conflict",
                ce.Message,
                (object?)null),

            DomainException de => (
                StatusCodes.Status422UnprocessableEntity,
                "domain_rule_violation",
                de.Message,
                (object?)null),

            _ => (
                StatusCodes.Status500InternalServerError,
                "server_error",
                "An unexpected error occurred.",
                (object?)null)
        };

        context.Response.StatusCode = statusCode;
        context.Response.ContentType = "application/json";

        var response = new
        {
            error = errorCode,
            message,
            errors,
            traceId = Activity.Current?.Id ?? context.TraceIdentifier
        };

        await context.Response.WriteAsJsonAsync(response);
    }
}

// Extension method for clean registration in Program.cs
public static class GlobalExceptionMiddlewareExtensions
{
    public static IApplicationBuilder UseGlobalExceptionHandler(
        this IApplicationBuilder app) =>
        app.UseMiddleware<GlobalExceptionMiddleware>();
}
```

---

## Middleware Registration Order

Order matters in ASP.NET Core middleware. Register in `Program.cs` as follows:

```csharp
app.UseCorrelationId();           // first — so all subsequent logs have correlation ID
app.UseGlobalExceptionHandler();  // early — catches exceptions from all subsequent middleware
app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
```

---

## Important Notes

- `GlobalExceptionMiddleware` must be registered **before** controllers. If registered after, exceptions from controllers will not be caught.
- Never return stack traces in production responses. The `traceId` is enough for correlation with server-side logs.
- In Java/Spring, global exception handling is done via `@ControllerAdvice` with `@ExceptionHandler` methods, which is equivalent in purpose. Middleware-level cross-cutting concerns (logging, correlation IDs) use `OncePerRequestFilter` implementations.
