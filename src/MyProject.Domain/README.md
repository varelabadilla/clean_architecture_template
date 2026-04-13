# MyProject.Domain

## Why This Project Exists

The Domain project is the **heart of the application**. It contains the enterprise business rules — the rules that exist regardless of whether the application is a web API, a CLI, a background worker, or a desktop app. These rules would exist even if there were no software.

In Clean Architecture terms, this is the **innermost circle**. It depends on nothing except SharedKernel. No frameworks, no ORM, no HTTP, no logging libraries. Just pure business logic expressed in code.

This isolation is not academic. It means:

- Domain logic can be unit tested without spinning up databases or web servers
- Business rules cannot be accidentally broken by infrastructure changes
- A developer reading Domain code sees only business intent, not technical noise

---

## What Belongs Here

- **Entities:** Objects with identity that persist over time and can change state
- **Aggregate Roots:** Entities that control access to a cluster of related objects
- **Value Objects:** Immutable objects defined entirely by their attributes
- **Domain Events:** Facts that happened within the domain
- **Domain Exceptions:** Violations of business invariants
- **Repository Contracts:** Interfaces that define persistence behavior (not implementations)
- **Specifications:** Encapsulated query/business criteria
- **Enums:** Domain-specific enumerated types
- **Domain Services:** Stateless operations that involve multiple aggregates

---

## What Does NOT Belong Here

- EF Core annotations (`[Table]`, `[Column]`, `[Key]`) — those go in Infrastructure/Persistence/Configurations
- MediatR, AutoMapper, or any application framework references
- DTOs or response models
- Repository implementations
- Any `using` statement that references Infrastructure, Application, or Api namespaces
- HTTP or web concepts

---

## Dependency Rule

```plaintext
MyProject.Domain → MyProject.SharedKernel (only)
```

---

## Recommended Naming Conventions

| Type | Convention | Example |
| --- | --- | --- |
| Entities | PascalCase noun, no suffix | `Order`, `Customer`, `Product` |
| Aggregate roots | Same as entities | `Order` (if it is the root) |
| Value objects | Noun describing the concept | `Money`, `Address`, `EmailAddress` |
| Domain events | Past tense + "Event" suffix | `OrderPlacedEvent`, `CustomerRegisteredEvent` |
| Domain exceptions | Descriptive + "Exception" suffix | `InsufficientStockException` |
| Repository interfaces | `I` + AggregateName + `Repository` | `IOrderRepository` |
| Specifications | Descriptive + "Specification" | `ActiveOrdersSpecification` |
| Enums | PascalCase singular | `OrderStatus`, `PaymentMethod` |
| Domain services | Noun + "Service" or "DomainService" | `PricingService`, `InventoryDomainService` |

---

## Key Design Principles

### Aggregates enforce their own invariants

An aggregate must never be in an invalid state. Constructors and methods should validate and throw `DomainException` if a rule is violated.

```csharp
public class Order : AggregateRoot
{
    private readonly List<OrderItem> _items = new();

    public Order(Guid id, CustomerId customerId)
    {
        Id = Guard.AgainstNull(id, nameof(id));
        CustomerId = Guard.AgainstNull(customerId, nameof(customerId));
        Status = OrderStatus.Pending;
    }

    public void AddItem(ProductId productId, int quantity, Money unitPrice)
    {
        if (Status != OrderStatus.Pending)
            throw new InvalidOrderStateException(Id, Status, "add items");

        _items.Add(new OrderItem(productId, quantity, unitPrice));
        AddDomainEvent(new OrderItemAddedEvent(Id, productId, quantity));
    }
}
```

### Value objects express domain concepts

Value objects model concepts that are defined entirely by their attributes. Two `Money` objects with the same amount and currency are identical.

```csharp
public record Money(decimal Amount, string Currency)
{
    public static Money operator +(Money a, Money b)
    {
        if (a.Currency != b.Currency)
            throw new CurrencyMismatchException(a.Currency, b.Currency);
        return new Money(a.Amount + b.Amount, a.Currency);
    }
}
```

### Domain events capture what happened

Raise domain events inside aggregates when something meaningful happens. Do not publish them immediately — let the infrastructure layer dispatch them after persistence.

---

## Folder Overview

| Folder | Contents |
| --- | --- |
| `Entities/` | Aggregate roots and child entities |
| `ValueObjects/` | Immutable domain concepts |
| `Enums/` | Domain-specific enumerated types |
| `Events/` | Domain event classes + `IDomainEventDispatcher` contract |
| `Exceptions/` | Domain-specific exception classes |
| `Specifications/` | `ISpecification<T>` contract + concrete specifications |
| `Repositories/` | Repository interface contracts |
| `Common/` | Base classes: `Entity<T>`, `AggregateRoot`, `ValueObject` |

---

## Important Notes

- Prefer **rich domain models** over anemic ones. If your entities are just bags of properties with no behavior, business logic has leaked into handlers or services — which makes it untestable and scattered.
- Domain services should be the exception, not the rule. If behavior can live on an entity or value object, it should.
- Never add `async/await` to domain methods. Domain logic is synchronous. Async concerns belong in Application handlers or Infrastructure.
