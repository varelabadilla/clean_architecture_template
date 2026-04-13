# Domain / Entities

## Why This Folder Exists

Entities are **objects with identity** — they are defined not by their attributes but by a thread of continuity and identity that runs through time. Two entities with the same data but different IDs are different objects. An `Order` from customer A and an `Order` from customer B with the same items are not the same order.

This folder contains the **aggregate roots and child entities** that model the core business concepts of your domain.

---

## What Belongs Here

- **Aggregate root classes:** The public entry points into a cluster of domain objects
- **Child entity classes:** Entities that belong to and are accessed through an aggregate root
- Domain behavior methods on those entities
- Internal invariant enforcement

---

## What Does NOT Belong Here

- Value objects (those go in `ValueObjects/`)
- Entities that are only used for read models (those belong in Infrastructure or Application)
- Entities decorated with ORM attributes (EF/JPA annotations stay in Infrastructure)
- Anemic classes with only getters and setters and no behavior

---

## Recommended Naming Conventions

| Type | Convention | Example |
| --- | --- | --- |
| Aggregate root | PascalCase noun | `Order`, `Customer`, `Invoice` |
| Child entity | PascalCase noun, implies ownership | `OrderItem`, `InvoiceLine` |
| Identity type | AggregateName + `Id` | `OrderId`, `CustomerId` |
| Constructor | Validates all required invariants | N/A |
| Methods | Verb phrase expressing intent | `Place()`, `Cancel()`, `AddItem()` |

---

## Aggregate Root Structure

```csharp
// .NET example
public class Order : AggregateRoot<OrderId>
{
    // Private setters — state changes only through methods
    public CustomerId CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    public Money TotalAmount { get; private set; }

    private readonly List<OrderItem> _items = new();
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();

    // Constructor enforces creation invariants
    public Order(OrderId id, CustomerId customerId)
    {
        Id = Guard.AgainstNull(id, nameof(id));
        CustomerId = Guard.AgainstNull(customerId, nameof(customerId));
        Status = OrderStatus.Pending;
        TotalAmount = Money.Zero("USD");

        AddDomainEvent(new OrderCreatedEvent(Id, CustomerId));
    }

    // Methods enforce state transition invariants
    public void AddItem(ProductId productId, int quantity, Money unitPrice)
    {
        if (Status != OrderStatus.Pending)
            throw new InvalidOrderStateException(Id, Status, "add items");

        var item = new OrderItem(productId, quantity, unitPrice);
        _items.Add(item);
        TotalAmount += item.Subtotal;

        AddDomainEvent(new OrderItemAddedEvent(Id, productId, quantity));
    }

    public void Place()
    {
        if (!_items.Any())
            throw new EmptyOrderException(Id);

        if (Status != OrderStatus.Pending)
            throw new InvalidOrderStateException(Id, Status, "place");

        Status = OrderStatus.Placed;
        AddDomainEvent(new OrderPlacedEvent(Id, CustomerId, TotalAmount));
    }

    public void Cancel(string reason)
    {
        if (Status is OrderStatus.Shipped or OrderStatus.Delivered)
            throw new InvalidOrderStateException(Id, Status, "cancel");

        Status = OrderStatus.Cancelled;
        AddDomainEvent(new OrderCancelledEvent(Id, reason));
    }
}
```

### Child Entity

```csharp
public class OrderItem : Entity<OrderItemId>
{
    public ProductId ProductId { get; private set; }
    public int Quantity { get; private set; }
    public Money UnitPrice { get; private set; }
    public Money Subtotal => UnitPrice * Quantity;

    internal OrderItem(ProductId productId, int quantity, Money unitPrice)
    {
        ProductId = Guard.AgainstNull(productId, nameof(productId));
        Quantity = Guard.AgainstNegativeOrZero(quantity, nameof(quantity));
        UnitPrice = Guard.AgainstNull(unitPrice, nameof(unitPrice));
        Id = OrderItemId.New();
    }
}
```

Note that `OrderItem`'s constructor is `internal` — it can only be created by `Order`. This enforces aggregate boundary rules at the compiler level.

---

## Aggregate Design Guidelines

- **One repository per aggregate root.** Never create a repository for a child entity.
- **Access child entities only through the root.** `order.Items` — not a separate `OrderItemRepository`.
- **Keep aggregates small.** An aggregate that spans more than 3-5 concepts is likely too large and will cause concurrency and performance issues.
- **Aggregates reference other aggregates by ID only.** `Order` holds a `CustomerId`, not a `Customer` reference. Cross-aggregate navigation is done in Application handlers, not in the domain model.

---

## Identity Type Pattern

Wrapping IDs in strongly-typed value objects prevents identity confusion (`CustomerId` cannot be passed where `OrderId` is expected):

```csharp
public record OrderId(Guid Value)
{
    public static OrderId New() => new(Guid.NewGuid());
    public static OrderId From(Guid value) => new(value);
    public override string ToString() => Value.ToString();
}
```

---

## Important Notes

- Every property should have a `private set`. State must only change through explicit methods that enforce invariants. Public setters indicate an anemic model.
- Domain events should be raised inside the aggregate methods, not in Application handlers. The aggregate knows when something meaningful happened.
- In Java/Spring, do not use Lombok `@Data` on aggregates — it generates public setters for all fields. Use `@Getter` only, and write explicit behavior methods.
