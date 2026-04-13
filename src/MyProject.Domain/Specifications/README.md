# Domain / Specifications

## Why This Folder Exists

Specifications encapsulate **business criteria** — the rules that determine whether a domain object satisfies a condition. Instead of scattering `Where(o => o.Status == OrderStatus.Placed && o.CreatedAt > cutoff)` expressions across repositories, handlers, and services, specifications give that criteria a name, a place to live, and the ability to be composed and reused.

This folder contains the `ISpecification<T>` contract (the base interface all specifications implement) and the concrete specification classes for each aggregate.

---

## What Belongs Here

- The `ISpecification<T>` base interface
- Concrete specifications for domain query criteria
- Composite specifications (`AndSpecification`, `OrSpecification`, `NotSpecification`)

---

## What Does NOT Belong Here

- Query logic that doesn't represent a reusable domain concept
- ORM-specific query builders (IQueryable extensions belong in Infrastructure)
- Application-level filtering logic (pagination, sorting)

---

## Recommended Naming Conventions

| Type | Convention | Example |
| --- | --- | --- |
| Base interface | `ISpecification<T>` | Fixed name |
| Concrete specification | Descriptive phrase + `Specification` | `ActiveOrdersSpecification`, `OrdersByCustomerSpecification` |
| Composite specs | `And/Or/Not` + `Specification` | `AndSpecification<T>` |

---

## `ISpecification<T>` Contract

```csharp
// .NET
public interface ISpecification<T>
{
    Expression<Func<T, bool>> Criteria { get; }
    List<Expression<Func<T, object>>> Includes { get; }
    List<string> IncludeStrings { get; }
    Expression<Func<T, object>>? OrderBy { get; }
    Expression<Func<T, object>>? OrderByDescending { get; }
    bool IsSatisfiedBy(T entity);
}
```

```java
// Java
public interface Specification<T> {
    boolean isSatisfiedBy(T entity);
    Specification<T> and(Specification<T> other);
    Specification<T> or(Specification<T> other);
    Specification<T> not();
    Predicate toPredicate(Root<T> root, CriteriaQuery<?> query, CriteriaBuilder cb); // JPA
}
```

---

## Base Specification Implementation

```csharp
// .NET base class
public abstract class Specification<T> : ISpecification<T>
{
    public abstract Expression<Func<T, bool>> Criteria { get; }
    public List<Expression<Func<T, object>>> Includes { get; } = new();
    public List<string> IncludeStrings { get; } = new();
    public Expression<Func<T, object>>? OrderBy { get; protected set; }
    public Expression<Func<T, object>>? OrderByDescending { get; protected set; }

    public bool IsSatisfiedBy(T entity) => Criteria.Compile()(entity);

    public Specification<T> And(Specification<T> other) =>
        new AndSpecification<T>(this, other);

    public Specification<T> Or(Specification<T> other) =>
        new OrSpecification<T>(this, other);

    protected void AddInclude(Expression<Func<T, object>> includeExpression) =>
        Includes.Add(includeExpression);
}
```

---

## Concrete Specification Examples

```csharp
// Single-condition specification
public class PlacedOrdersSpecification : Specification<Order>
{
    public override Expression<Func<Order, bool>> Criteria =>
        order => order.Status == OrderStatus.Placed;
}

// Parameterized specification
public class OrdersByCustomerSpecification : Specification<Order>
{
    private readonly CustomerId _customerId;

    public OrdersByCustomerSpecification(CustomerId customerId)
    {
        _customerId = customerId;
        AddInclude(o => o.Items);
    }

    public override Expression<Func<Order, bool>> Criteria =>
        order => order.CustomerId == _customerId;
}

// Date-range specification
public class RecentOrdersSpecification : Specification<Order>
{
    private readonly DateTime _cutoff;

    public RecentOrdersSpecification(int daysBack = 30)
    {
        _cutoff = DateTime.UtcNow.AddDays(-daysBack);
    }

    public override Expression<Func<Order, bool>> Criteria =>
        order => order.CreatedAt >= _cutoff;
}
```

### Composition

```csharp
// In a handler or repository
var spec = new OrdersByCustomerSpecification(customerId)
    .And(new PlacedOrdersSpecification())
    .And(new RecentOrdersSpecification(60));

var orders = await _orderRepository.FindAsync(spec);
```

---

## Two Uses of Specifications

### 1. In-memory (domain logic)

Use `IsSatisfiedBy(entity)` to check business criteria against an already-loaded object:

```csharp
if (!eligibilitySpec.IsSatisfiedBy(customer))
    throw new CustomerNotEligibleException(customer.Id);
```

### 2. Query (persistence)

Pass specifications to repositories so they can be translated to SQL:

```csharp
// IOrderRepository contract
Task<IReadOnlyList<Order>> FindAsync(ISpecification<Order> specification);
```

---

## Important Notes

- Specifications express **business intent** in their names. `OrdersEligibleForPromotion` is better than `OrdersWhereStatusIsActiveAndTotalGreaterThan100`.
- Avoid specifications that are too granular (one per condition) or too broad (one that accepts raw expressions). The right granularity is a reusable business concept.
- The `IQueryable` translation layer (applying `Criteria` to an EF `DbSet`) lives in `Infrastructure/Persistence/Repositories/`, not here. The specification is pure business logic; the ORM translation is infrastructure.
- In Java, specifications can be implemented using the JPA `Specification` interface from Spring Data, or as plain `Predicate<T>` for in-memory use.
