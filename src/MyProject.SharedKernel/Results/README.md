# SharedKernel / Results

## Why This Folder Exists

This folder contains the **Result pattern** types used throughout the solution to model expected success and failure outcomes without relying on exceptions as control flow.

Exceptions should be reserved for truly exceptional situations — unexpected runtime failures, not predictable business outcomes like "order not found" or "validation failed." Using `Result<T>` makes the contract of every method explicit: callers are forced to handle both success and failure paths.

This is a cross-cutting concern because handlers, domain services, and API controllers all need to produce and consume results in a consistent way.

---

## What Belongs Here

- `Result` — non-generic result for operations with no return value
- `Result<T>` — generic result wrapping a value on success
- `Error` — immutable, serializable error descriptor
- `ErrorType` — enum categorizing error semantics
- `PagedList<T>` — pagination wrapper for query results
- `PaginationParams` — input model for paginated queries

---

## What Does NOT Belong Here

- Exception classes (those go in `Domain/Exceptions/` or `Application/Exceptions/`)
- Domain-specific error codes (define those as static `Error` constants inside each domain or feature)
- HTTP status code mapping (that belongs in `Api/Middleware/`)

---

## Recommended File Names

```plaintext
Result.cs
Result<T>.cs          // or combined in Result.cs as two classes
Error.cs
ErrorType.cs
PagedList.cs
PaginationParams.cs
ValidationError.cs    // optional: wraps multiple field-level errors
```

---

## Recommended Implementations

### ErrorType enum

```csharp
public enum ErrorType
{
    Failure,        // generic unclassified failure
    Validation,     // input did not pass validation rules
    NotFound,       // requested resource does not exist
    Conflict,       // state conflict (e.g. duplicate, concurrent edit)
    Unauthorized,   // identity not authenticated
    Forbidden       // identity authenticated but not authorized
}
```

### Error record

```csharp
public record Error(string Code, string Description, ErrorType Type)
{
    public static readonly Error None =
        new(string.Empty, string.Empty, ErrorType.Failure);

    public static Error NotFound(string code, string description) =>
        new(code, description, ErrorType.NotFound);

    public static Error Validation(string code, string description) =>
        new(code, description, ErrorType.Validation);
}
```

### `Result<T>`

```csharp
public class Result<T>
{
    private Result(bool isSuccess, T? value, Error error)
    {
        IsSuccess = isSuccess;
        Value = value;
        Error = error;
    }

    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;
    public T? Value { get; }
    public Error Error { get; }

    public static Result<T> Success(T value) =>
        new(true, value, Error.None);

    public static Result<T> Failure(Error error) =>
        new(false, default, error);
}
```

### `PagedList<T>`

```csharp
public class PagedList<T>
{
    public IReadOnlyList<T> Items { get; }
    public int Page { get; }
    public int PageSize { get; }
    public int TotalCount { get; }
    public bool HasNextPage => Page * PageSize < TotalCount;
    public bool HasPreviousPage => Page > 1;
}
```

---

## Error Code Conventions

Error codes should be **dot-separated namespaced strings** that identify the origin and nature of the failure:

```plaintext
"Order.NotFound"
"Order.AlreadyCancelled"
"Customer.EmailAlreadyRegistered"
"Validation.RequiredField"
```

This format is readable, serializable, and can be used as keys for i18n lookups on the client.

---

## Important Notes

- Define domain-specific `Error` constants as static members of the relevant class, not here. For example, `OrderErrors.NotFound` lives in the Domain or Application layer, and references `Error.NotFound(...)` from this folder.
- `Result<T>` is not a replacement for all exceptions. Null reference errors, programming mistakes, and infrastructure failures should still throw — they are not expected flows.
- In Java, this pattern is commonly implemented using sealed interfaces or a similar discriminated union pattern, or with libraries like `vavr`'s `Either<L,R>`.
