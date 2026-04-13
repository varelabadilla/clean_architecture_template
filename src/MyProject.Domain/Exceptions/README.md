# Domain / Exceptions

## Why This Folder Exists

Domain exceptions represent **violations of business invariants** — situations where the domain model has been asked to do something that is fundamentally illegal according to business rules. These exceptions exist because the domain has a language for things going wrong, and that language belongs here, not scattered across application handlers or infrastructure code.

The key distinction from Application exceptions is **who throws them**: entities and aggregates throw domain exceptions. Application handlers throw application exceptions. This folder belongs to entities.

---

## What Belongs Here

- A `DomainException` base class
- Exceptions thrown by entities when invariants are violated
- Exceptions thrown by value objects when construction constraints fail
- Exceptions thrown by domain services for rule violations

---

## What Does NOT Belong Here

- `NotFoundException` — that is an application flow concern, not a domain invariant
- `ValidationException` from FluentValidation — that is an application input concern
- HTTP-related exceptions
- Infrastructure exceptions (database errors, connection failures)

---

## The Distinction That Matters

| Exception type | Means | Lives in |
| --- | --- | --- |
| `InsufficientStockException` | Business rule: cannot sell what you don't have | `Domain/Exceptions/` |
| `InvalidOrderStateException` | Business rule: cannot ship a cancelled order | `Domain/Exceptions/` |
| `NotFoundException` | Application flow: requested resource doesn't exist | `Application/Exceptions/` |
| `ValidationException` | Application input: request failed validation | `Application/Exceptions/` |
| `DbUpdateException` | Infrastructure failure | Never bubble up — wrap it |

---

## Recommended Naming Conventions

| Type | Convention | Example |
| --- | --- | --- |
| Base class | `DomainException` | Fixed name |
| Specific exceptions | Descriptive noun phrase + `Exception` | `InsufficientStockException` |
| Message | Include entity id and context | `"Order {id} cannot be shipped: status is {status}"` |

---

## Recommended Implementations

### Base class

```csharp
// .NET
public abstract class DomainException : Exception
{
    protected DomainException(string message) : base(message) { }

    protected DomainException(string message, Exception innerException)
        : base(message, innerException) { }
}
```

```java
// Java
public abstract class DomainException extends RuntimeException {
    protected DomainException(String message) {
        super(message);
    }
}
```

### Concrete domain exceptions

```csharp
public class InvalidOrderStateException : DomainException
{
    public InvalidOrderStateException(OrderId orderId, OrderStatus currentStatus, string attemptedOperation)
        : base($"Order '{orderId}' cannot perform '{attemptedOperation}' while in status '{currentStatus}'.") { }
}

public class InsufficientStockException : DomainException
{
    public InsufficientStockException(ProductId productId, int requested, int available)
        : base($"Product '{productId}' has insufficient stock. Requested: {requested}, Available: {available}.") { }
}

public class CurrencyMismatchException : DomainException
{
    public CurrencyMismatchException(string currencyA, string currencyB)
        : base($"Cannot operate on money values with different currencies: '{currencyA}' and '{currencyB}'.") { }
}

public class EmptyOrderException : DomainException
{
    public EmptyOrderException(OrderId orderId)
        : base($"Order '{orderId}' cannot be placed because it contains no items.") { }
}
```

---

## Usage in Aggregate

```csharp
public void Ship(TrackingNumber trackingNumber)
{
    if (Status != OrderStatus.Confirmed)
        throw new InvalidOrderStateException(Id, Status, "ship");

    TrackingNumber = trackingNumber;
    Status = OrderStatus.Shipped;
    AddDomainEvent(new OrderShippedEvent(Id, trackingNumber));
}
```

---

## HTTP Translation

Domain exceptions are translated to HTTP responses in the API layer, never here. The `GlobalExceptionMiddleware` in `Api/Middleware/` catches `DomainException` and maps it to a `422 Unprocessable Entity` or `409 Conflict` response, depending on context.

```csharp
// Api/Middleware/GlobalExceptionMiddleware.cs
case DomainException domainEx:
    context.Response.StatusCode = StatusCodes.Status422UnprocessableEntity;
    await WriteErrorResponse(context, "domain_rule_violation", domainEx.Message);
    break;
```

---

## Important Notes

- Domain exceptions use `RuntimeException` in Java (unchecked) — callers should not be forced to handle them with `try/catch` everywhere. They represent programmer-level invariant violations or unexpected business state.
- Include meaningful context in exception messages. The entity ID, the invalid state, and the attempted operation together make the error immediately diagnosable without additional logging.
- Do not catch and swallow domain exceptions in Application handlers. Let them propagate to the middleware. Catching them just to rethrow a different exception adds noise without value.
