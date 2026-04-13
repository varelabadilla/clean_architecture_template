# Application / Common

## Why This Folder Exists

This folder contains **shared building blocks** used across the Application layer that do not belong to any specific feature. These are the marker interfaces, base classes, and shared utilities that give the CQRS pipeline its structure and type safety.

---

## What Belongs Here

- Marker interfaces for commands and queries (`ICommand`, `IQuery<T>`)
- Common base request/response types
- Application-level constants
- Shared utilities used only within the Application layer

---

## What Does NOT Belong Here

- Business logic
- Feature-specific types
- Infrastructure types
- Domain types

---

## Recommended File Names

```plaintext
ICommand.cs
IQuery.cs
ICacheableQuery.cs
BaseHandler.cs          // optional: shared handler behavior
```

---

## Core Marker Interfaces

These interfaces give MediatR request types semantic meaning and allow behaviors to apply selectively:

```csharp
// Commands — write operations, return Result
public interface ICommand : IRequest<Result> { }
public interface ICommand<TResponse> : IRequest<Result<TResponse>> { }

// Queries — read operations, return Result<T>
public interface IQuery<TResponse> : IRequest<Result<TResponse>> { }

// Cacheable queries — signals CachingBehavior to apply
public interface ICacheableQuery<TResponse> : IQuery<TResponse>
{
    string CacheKey { get; }
    TimeSpan? CacheDuration { get; }
}
```

### Usage

```csharp
// A command that returns the new entity's ID
public record PlaceOrderCommand(...) : ICommand<Guid>;

// A command with no return value
public record CancelOrderCommand(...) : ICommand;

// A query
public record GetOrderByIdQuery(Guid OrderId) : IQuery<OrderDetailDto>;

// A cacheable query
public record GetProductCatalogQuery() : ICacheableQuery<IReadOnlyList<ProductDto>>
{
    public string CacheKey => "product-catalog";
    public TimeSpan? CacheDuration => TimeSpan.FromMinutes(10);
}
```

---

## Important Notes

- These marker interfaces are what allow `TransactionBehavior` to apply only to commands and `CachingBehavior` to apply only to cacheable queries — without inspecting type names at runtime.
- Keep this folder minimal. Resist adding application-wide helpers that blur responsibility. If something is needed by all layers, it likely belongs in `SharedKernel`.
