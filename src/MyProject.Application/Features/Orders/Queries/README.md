# Features / Orders / Queries

## Purpose

Contains the **input models for read operations** on the Order aggregate. Each query represents a request to retrieve data without changing any state.

## Naming Convention

`{Get|List|Find}{Resource}{OptionalFilter}Query`

```
GetOrderByIdQuery
ListOrdersByCustomerQuery
GetOrderStatusQuery
FindRecentOrdersQuery
```

## Rules

- Queries are **immutable** — use `record` types
- Queries implement `IQuery<TResponse>` where `TResponse` is a DTO or `PagedResult<TDto>`
- Queries **never mutate state** — they are side-effect free
- One file per query
- Query response types are DTOs, never domain entities

## Cacheable Queries

If a query result can be cached, implement `ICacheableQuery<TResponse>` and declare a `CacheKey` and `CacheDuration`. The `CachingBehavior` in the pipeline handles caching automatically:

```csharp
public record GetProductCatalogQuery() : ICacheableQuery<IReadOnlyList<ProductDto>>
{
    public string CacheKey => "product:catalog";
    public TimeSpan? CacheDuration => TimeSpan.FromMinutes(15);
}
```

## Example

```csharp
// GetOrderByIdQuery.cs
public record GetOrderByIdQuery(Guid OrderId) : IQuery<OrderDetailDto>;

// ListOrdersByCustomerQuery.cs
public record ListOrdersByCustomerQuery(
    Guid CustomerId,
    int Page = 1,
    int PageSize = 20) : IQuery<PagedResult<OrderSummaryDto>>;
```

## Example files

```
GetOrderByIdQuery.cs
ListOrdersByCustomerQuery.cs
GetOrderStatusQuery.cs
ListRecentOrdersQuery.cs
```
