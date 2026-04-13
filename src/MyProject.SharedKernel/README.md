# MyProject.SharedKernel

## Why This Project Exists

The SharedKernel solves a specific problem: every layer in a Clean Architecture needs a common vocabulary — base types, result wrappers, guard clauses — but none of them should be the "owner" of that vocabulary, because that would create an upward dependency.

SharedKernel is the **foundational layer with no dependencies**. It is not a utility dumping ground. It contains only the primitive building blocks that all layers agree to share, without any business logic, framework imports, or infrastructure concerns.

In DDD terms, a Shared Kernel is a small subset of the domain model that multiple bounded contexts agree to share. In this structural context, it extends that concept to include architectural primitives.

**Dependency position:** No project depends on SharedKernel **except** every other project. SharedKernel depends on nothing.

---

## What Belongs Here

- **Base abstractions for domain modeling:** `IEntity<TId>`, `IAggregateRoot`, `IValueObject`, `IDomainEvent`
- **Result pattern types:** `Result<T>`, `Result`, `Error`, `ErrorType`
- **Paged collection wrappers:** `PagedList<T>`, `PaginationParams`
- **Guard clauses:** Static helpers to validate preconditions (`Guard.AgainstNull`, `Guard.AgainstNegative`)
- **Extension methods:** Pure extension methods with no external dependencies (string, datetime, enumerable helpers)
- **Base exceptions:** A `DomainException` base class if you want a hierarchy, or a generic `AppException`

---

## What Does NOT Belong Here

- Business logic of any kind
- Framework-specific code (no EF, no MediatR, no ASP.NET references)
- DTOs or ViewModels
- Application services or interfaces
- Anything that requires a NuGet package beyond the base runtime
- Infrastructure concerns (logging implementations, caching, etc.)

> **Rule of thumb:** If adding something here requires installing a NuGet package, it probably does not belong here.

---

## Recommended Naming Conventions

| Type | Convention | Example |
| --- | --- | --- |
| Interfaces | `I` prefix + descriptive noun | `IEntity`, `IAggregateRoot` |
| Base classes | Descriptive noun, no prefix | `ValueObject`, `AggregateRoot` |
| Result types | Noun describing the concept | `Result`, `Error`, `PagedList` |
| Guard class | `Guard` (static class) | `Guard` |
| Enums | PascalCase, singular | `ErrorType` |
| Extension classes | `{Type}Extensions` | `StringExtensions`, `DateTimeExtensions` |

---

## Recommended Types Per Folder

### Abstractions/

```plaintext
IEntity.cs              // base identity contract
IEntity<TId>.cs         // generic version with typed Id
IAggregateRoot.cs       // marker interface for aggregates
IValueObject.cs         // marker interface for value objects
IDomainEvent.cs         // base contract for domain events
IAuditableEntity.cs     // CreatedAt, UpdatedAt, CreatedBy
ISoftDeletable.cs       // IsDeleted, DeletedAt
```

### Results/

```plaintext
Result.cs               // non-generic result (success/failure)
Result<T>.cs            // generic result with value
Error.cs                // immutable error descriptor (Code, Description, Type)
ErrorType.cs            // enum: Validation, NotFound, Conflict, Failure, Unauthorized
PagedList<T>.cs         // pagination wrapper with metadata
PaginationParams.cs     // page number + page size input
```

### Guards/

```plaintext
Guard.cs                // static class with all guard methods
GuardClause.cs          // fluent chain entry point (optional)
```

### Extensions/

```plaintext
StringExtensions.cs
DateTimeExtensions.cs
EnumerableExtensions.cs
```

---

## Result Pattern — Key Design Decision

Using `Result<T>` instead of throwing exceptions for expected failure flows is a deliberate choice. Exceptions are for **exceptional, unexpected situations**. Business rule violations (a user not found, a validation failure) are expected flows and should be modeled as data.

```csharp
// .NET example
public class Result<T>
{
    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;
    public T Value { get; }
    public Error Error { get; }

    public static Result<T> Success(T value) => new(true, value, Error.None);
    public static Result<T> Failure(Error error) => new(false, default, error);
}

public record Error(string Code, string Description, ErrorType Type)
{
    public static readonly Error None = new(string.Empty, string.Empty, ErrorType.Failure);
}
```

```java
// Java equivalent
public class Result<T> {
    private final boolean success;
    private final T value;
    private final AppError error;

    public static <T> Result<T> success(T value) { ... }
    public static <T> Result<T> failure(AppError error) { ... }
}
```

---

## Important Notes

- Keep this project **extremely lean**. Resist the temptation to add convenience utilities. Every addition increases the coupling surface for all other layers.
- This project should rarely change. It represents the contract everyone has agreed on. Breaking changes here propagate to every layer.
- In a multi-module or microservices setup, SharedKernel is typically distributed as an internal NuGet/Maven package so all services share the same primitives.
- Do not version SharedKernel independently from the solution unless you have a strong reason to. Keeping it in the same repository avoids synchronization overhead.
