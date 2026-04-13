# Infrastructure / Persistence / Repositories

## Why This Folder Exists

This folder contains the **concrete implementations of the repository interfaces** defined in `Domain/Repositories/`. While the Domain layer declares what persistence must be capable of, this folder provides the actual EF Core (or Dapper) implementation of those capabilities.

The separation ensures that application and domain code depends on abstractions, while the concrete database interaction is fully contained here and swappable.

---

## What Belongs Here

- One repository implementation class per aggregate root
- A shared `BaseRepository<T>` if common behavior warrants it
- Specification evaluator helper (translates `ISpecification<T>` to `IQueryable<T>`)

---

## What Does NOT Belong Here

- Repository interfaces (those live in `Domain/Repositories/`)
- Business logic of any kind
- Direct SQL that belongs in a query service (complex reporting queries should use a separate read model)
- Generic repository base classes that expose `IQueryable<T>` publicly

---

## Recommended Naming Conventions

| Type | Convention | Example |
| --- | --- | --- |
| Implementation class | AggregateName + `Repository` | `OrderRepository`, `CustomerRepository` |
| Base class (optional) | `BaseRepository<T>` | Fixed name |
| Specification evaluator | `SpecificationEvaluator<T>` | Fixed name |

---

## Repository Implementation Example

```csharp
public class OrderRepository : IOrderRepository
{
    private readonly AppDbContext _context;

    public OrderRepository(AppDbContext context) => _context = context;

    public async Task<Order?> GetByIdAsync(
        OrderId id,
        CancellationToken cancellationToken = default)
    {
        return await _context.Orders
            .Include(o => o.Items)
            .FirstOrDefaultAsync(o => o.Id == id, cancellationToken);
    }

    public async Task<IReadOnlyList<Order>> FindAsync(
        ISpecification<Order> specification,
        CancellationToken cancellationToken = default)
    {
        return await SpecificationEvaluator<Order>
            .GetQuery(_context.Orders.AsQueryable(), specification)
            .ToListAsync(cancellationToken);
    }

    public async Task<bool> ExistsAsync(
        OrderId id,
        CancellationToken cancellationToken = default)
    {
        return await _context.Orders
            .AnyAsync(o => o.Id == id, cancellationToken);
    }

    public async Task AddAsync(
        Order order,
        CancellationToken cancellationToken = default)
    {
        await _context.Orders.AddAsync(order, cancellationToken);
    }

    public void Update(Order order) => _context.Orders.Update(order);

    public void Delete(Order order) => _context.Orders.Remove(order);
}
```

---

## Specification Evaluator

Translates `ISpecification<T>` into EF Core LINQ queries:

```csharp
public static class SpecificationEvaluator<T> where T : class
{
    public static IQueryable<T> GetQuery(
        IQueryable<T> inputQuery,
        ISpecification<T> specification)
    {
        var query = inputQuery;

        if (specification.Criteria is not null)
            query = query.Where(specification.Criteria);

        query = specification.Includes
            .Aggregate(query, (current, include) =>
                current.Include(include));

        query = specification.IncludeStrings
            .Aggregate(query, (current, include) =>
                current.Include(include));

        if (specification.OrderBy is not null)
            query = query.OrderBy(specification.OrderBy);

        if (specification.OrderByDescending is not null)
            query = query.OrderByDescending(specification.OrderByDescending);

        return query;
    }
}
```

---

## Important Notes

- Repositories do **not** call `SaveChangesAsync`. Persistence is committed by the Unit of Work (`AppDbContext.SaveChangesAsync`) at the handler level, coordinated by `TransactionBehavior`. This ensures atomic operations across multiple repository calls.
- `Add` and `Update` do not need to be async — EF Core tracks entities in memory. Only `SaveChangesAsync` hits the database.
- For complex reporting queries that don't map cleanly to aggregate retrieval, use a dedicated read service (a query service using raw SQL or Dapper) rather than shoehorning them into a repository.
- In Java/Spring, implement these classes as `@Repository`-annotated adapters that delegate to `JpaRepository` internally, while implementing the domain `OrderRepository` interface.
