# Application / Exceptions

## Why This Folder Exists

Application exceptions represent **expected failure conditions in use case flows** — situations that are predictable and that the system must handle gracefully. These are not programming errors and not domain invariant violations; they are the application saying "I cannot fulfill this request because of a known condition."

The distinction from Domain exceptions is about **who throws them and why**:

- Domain exceptions: thrown by entities when a business rule is violated
- Application exceptions: thrown by handlers when a use case cannot be completed due to flow conditions

---

## What Belongs Here

- `NotFoundException` — a required resource does not exist
- `ForbiddenException` — the authenticated user is not authorized for this action
- `ConflictException` — a state conflict prevents the operation (e.g., duplicate resource)
- `ValidationException` — input failed validation (may wrap FluentValidation failures)

---

## What Does NOT Belong Here

- Domain exceptions (invariant violations — those go in `Domain/Exceptions/`)
- Infrastructure exceptions (database errors, network failures — those are wrapped, not bubbled)
- HTTP-specific exceptions
- Exceptions with business logic in them

---

## Recommended Naming Conventions

| Type | Convention | Example |
| --- | --- | --- |
| Base class | `ApplicationException` | Avoids conflict with `System.ApplicationException` — use a project-specific name |
| Specific exceptions | Descriptive + `Exception` | `NotFoundException`, `ForbiddenException` |

> **Note for .NET:** The name `ApplicationException` conflicts with `System.ApplicationException`. Name your base class `AppException` or prefix it: `MyProjectAppException`.

---

## Recommended Implementations

### Base class

```csharp
// .NET
public abstract class AppException : Exception
{
    public abstract int StatusCode { get; }

    protected AppException(string message) : base(message) { }
}
```

```java
// Java
public abstract class AppException extends RuntimeException {
    public abstract int getStatusCode();

    protected AppException(String message) {
        super(message);
    }
}
```

### NotFoundException

```csharp
public class NotFoundException : AppException
{
    public override int StatusCode => StatusCodes.Status404NotFound;

    public NotFoundException(string resourceName, object key)
        : base($"Resource '{resourceName}' with key '{key}' was not found.") { }

    // Convenience factory methods
    public static NotFoundException For<T>(object key) =>
        new(typeof(T).Name, key);
}
```

### ForbiddenException

```csharp
public class ForbiddenException : AppException
{
    public override int StatusCode => StatusCodes.Status403Forbidden;

    public ForbiddenException()
        : base("You do not have permission to perform this action.") { }

    public ForbiddenException(string message) : base(message) { }
}
```

### ConflictException

```csharp
public class ConflictException : AppException
{
    public override int StatusCode => StatusCodes.Status409Conflict;

    public ConflictException(string message) : base(message) { }
}
```

### ValidationException

```csharp
// Wraps FluentValidation failures into a structured exception
public class ValidationException : AppException
{
    public override int StatusCode => StatusCodes.Status422UnprocessableEntity;

    public IReadOnlyDictionary<string, string[]> Errors { get; }

    public ValidationException(IEnumerable<ValidationFailure> failures)
        : base("One or more validation errors occurred.")
    {
        Errors = failures
            .GroupBy(f => f.PropertyName, f => f.ErrorMessage)
            .ToDictionary(g => g.Key, g => g.ToArray());
    }
}
```

---

## Usage in Handlers

```csharp
public async Task<Result<OrderDetailDto>> Handle(
    GetOrderByIdQuery query, CancellationToken cancellationToken)
{
    var order = await _orders.GetByIdAsync(
        OrderId.From(query.OrderId), cancellationToken);

    if (order is null)
        throw new NotFoundException(nameof(Order), query.OrderId);

    // Authorization check
    if (order.CustomerId.Value != _currentUser.UserId)
        throw new ForbiddenException();

    return Result<OrderDetailDto>.Success(_mapper.Map<OrderDetailDto>(order));
}
```

---

## HTTP Mapping in the API Layer

These exceptions are translated to HTTP responses by `GlobalExceptionMiddleware` in `Api/Middleware/`:

```csharp
case NotFoundException notFoundEx:
    statusCode = StatusCodes.Status404NotFound;
    break;

case ForbiddenException forbiddenEx:
    statusCode = StatusCodes.Status403Forbidden;
    break;

case ValidationException validationEx:
    statusCode = StatusCodes.Status422UnprocessableEntity;
    // Include validation error details in response body
    break;
```

---

## Important Notes

- Prefer the `Result<T>` pattern over throwing exceptions for **predictable failures** when possible. Use exceptions for conditions that interrupt normal flow (a null where something was guaranteed to exist).
- When using `Result<T>` throughout, `NotFoundException` and similar exceptions become less common — the handler returns `Result.Failure(Error.NotFound(...))` instead of throwing. Both patterns are valid; choose one and be consistent.
- In Java, these exceptions extend `RuntimeException` (unchecked). Use Spring's `@ExceptionHandler` or `@ControllerAdvice` to translate them to HTTP responses.
