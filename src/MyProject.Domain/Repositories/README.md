# Domain / Repositories

## Why This Folder Exists

Repositories are the **contracts** that define how aggregates are persisted and retrieved. The Domain layer defines what persistence must be capable of — the Application layer calls it — and the Infrastructure layer implements it.

Defining repository interfaces here ensures that the Domain and Application layers can reason about persistence in domain terms, without ever knowing whether data lives in SQL Server, MongoDB, an in-memory store, or a mock for testing.

This folder contains **only interfaces**. No implementations.

---

## What Belongs Here

- One repository interface per aggregate root
- Methods expressed in domain language, not in persistence language
- Specification-based query methods
- Unit of Work interface (if used)

---

## What Does NOT Belong Here

- Repository implementations (those go in `Infrastructure/Persistence/Repositories/`)
- Read-only query services / projections (those belong in `Application/Abstractions/` or dedicated query services)
- Generic repository base interfaces (avoid them — they expose too much surface area)
- ORM types like `IQueryable<T>` or `DbSet<T>`

---

## Recommended Naming Conventions

| Type | Convention | Example |
| --- | --- | --- |
| Repository interface | `I` + AggregateName + `Repository` | `IOrderRepository`, `ICustomerRepository` |
| Methods — by id | `GetByIdAsync` | `GetByIdAsync(OrderId id)` |
| Methods — collections | `GetAllAsync`, `FindAsync` | `FindAsync(ISpecification<Order>)` |
| Methods — existence | `ExistsAsync` | `ExistsAsync(OrderId id)` |
| Methods — persistence | `AddAsync`, `UpdateAsync`, `DeleteAsync` | Standard names |

---

## Interface Structure

```csharp
// .NET
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(
        OrderId id,
        CancellationToken cancellationToken = default);

    Task<IReadOnlyList<Order>> FindAsync(
        ISpecification<Order> specification,
        CancellationToken cancellationToken = default);

    Task<bool> ExistsAsync(
        OrderId id,
        CancellationToken cancellationToken = default);

    Task AddAsync(
        Order order,
        CancellationToken cancellationToken = default);

    void Update(Order order);

    void Delete(Order order);
}
```

```java
// Java
public interface OrderRepository {
    Optional<Order> findById(OrderId id);
    List<Order> findAll(Specification<Order> specification);
    boolean existsById(OrderId id);
    Order save(Order order);
    void delete(Order order);
}
```

---

## Unit of Work Interface

If your application uses a Unit of Work pattern to coordinate multiple repositories in a single transaction:

```csharp
public interface IUnitOfWork
{
    IOrderRepository Orders { get; }
    ICustomerRepository Customers { get; }
    Task<int> SaveChangesAsync(CancellationToken cancellationToken = default);
}
```

This interface lives here in `Domain/Repositories/` alongside the individual repository contracts.

---

## What NOT to Include in Repository Interfaces

Avoid the **Generic Repository anti-pattern**:

```csharp
// AVOID: Too broad, exposes infrastructure-like surface
public interface IRepository<T>
{
    IQueryable<T> GetAll();      // IQueryable leaks EF concerns
    void Add(T entity);
    void Update(T entity);
    void Delete(T entity);
}
```

Instead, define explicit interfaces per aggregate with methods that reflect actual application needs:

```csharp
// PREFER: Explicit, domain-meaningful interface
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(OrderId id, CancellationToken ct = default);
    Task<IReadOnlyList<Order>> GetPendingOrdersAsync(CancellationToken ct = default);
    Task<IReadOnlyList<Order>> GetByCustomerAsync(CustomerId customerId, CancellationToken ct = default);
    Task AddAsync(Order order, CancellationToken ct = default);
    void Update(Order order);
}
```

---

## One Repository Per Aggregate Root

The rule is firm: **only aggregate roots get repositories**. `OrderItem` is a child entity of `Order`. You never retrieve an `OrderItem` independently — you go through its parent `Order`.

| Aggregate root | Has repository | Child entities | No repository |
| --- | --- | --- |---|
| `Order` | `IOrderRepository` | `OrderItem` | — |
| `Customer` | `ICustomerRepository` | `CustomerAddress` | — |
| `Product` | `IProductRepository` | `ProductVariant` | — |

---

## Important Notes

- Return `Order?` (nullable) from `GetByIdAsync`, not `Order`. Null signals absence. Throwing `NotFoundException` is the Application handler's responsibility after checking the null result.
- `CancellationToken` should be a parameter on all async methods to support cooperative cancellation.
- Do not add pagination or sorting parameters directly to repository interfaces. Prefer Specification objects that encapsulate that logic, keeping the interface stable when requirements change.
- In Java/Spring, `JpaRepository<Order, UUID>` from Spring Data is the infrastructure implementation. Define your own `OrderRepository` interface here in the domain and have the infrastructure adapter implement it.
