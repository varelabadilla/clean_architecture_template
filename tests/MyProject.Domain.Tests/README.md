# MyProject.Domain.Tests

## Why This Project Exists

Domain tests are **pure unit tests** — no database, no HTTP, no mocks of external services. They test the business logic encoded in entities, value objects, specifications, and domain services in complete isolation.

Because the Domain layer has zero infrastructure dependencies, these tests are fast (milliseconds each), stable (no network or I/O flakiness), and directly validate the rules that matter most: business invariants.

---

## What to Test Here

- **Entity invariants:** Valid and invalid state transitions, constructor validation
- **Aggregate behavior:** Methods that mutate state and raise domain events
- **Value object equality:** Two instances with same data are equal; mutation returns new instance
- **Value object validation:** Construction with invalid data throws `DomainException`
- **Domain events:** Correct events are raised when aggregate methods are called
- **Specifications:** `IsSatisfiedBy` returns correct result for given entity state
- **Domain services:** Business logic involving multiple aggregates or value objects

---

## What NOT to Test Here

- Repository behavior (that belongs in Infrastructure.Tests)
- Handler orchestration (that belongs in Application.Tests)
- HTTP contracts (that belongs in Api.Tests)
- Anything requiring a database or external service

---

## Recommended Project Structure

```plaintext
MyProject.Domain.Tests/
├─ Entities/
│  ├─ OrderTests.cs
│  └─ CustomerTests.cs
├─ ValueObjects/
│  ├─ MoneyTests.cs
│  └─ AddressTests.cs
├─ Specifications/
│  └─ PlacedOrdersSpecificationTests.cs
└─ Common/
   └─ Builders/                  // test data builders
      ├─ OrderBuilder.cs
      └─ CustomerBuilder.cs
```

---

## Naming Conventions

| Type | Convention | Example |
| --- | --- | --- |
| Test class | Class under test + `Tests` | `OrderTests`, `MoneyTests` |
| Test method | `{Method}_{Condition}_{ExpectedResult}` | `Place_WhenNoItems_ThrowsEmptyOrderException` |
| Builder class | Entity name + `Builder` | `OrderBuilder` |

---

## Test Examples

### Entity invariant test

```csharp
public class OrderTests
{
    [Fact]
    public void Place_WhenOrderHasItems_SetsStatusToPlaced()
    {
        // Arrange
        var order = OrderBuilder.New()
            .WithItem(ProductId.New(), 2, Money.Of(50, "USD"))
            .Build();

        // Act
        order.Place();

        // Assert
        order.Status.Should().Be(OrderStatus.Placed);
    }

    [Fact]
    public void Place_WhenOrderHasNoItems_ThrowsEmptyOrderException()
    {
        // Arrange
        var order = OrderBuilder.New().Build();

        // Act
        var act = () => order.Place();

        // Assert
        act.Should().Throw<EmptyOrderException>();
    }

    [Fact]
    public void Place_WhenOrderAlreadyPlaced_ThrowsInvalidOrderStateException()
    {
        // Arrange
        var order = OrderBuilder.New()
            .WithItem(ProductId.New(), 1, Money.Of(100, "USD"))
            .InStatus(OrderStatus.Placed)
            .Build();

        // Act
        var act = () => order.Place();

        // Assert
        act.Should().Throw<InvalidOrderStateException>();
    }

    [Fact]
    public void Place_WhenSuccessful_RaisesOrderPlacedEvent()
    {
        // Arrange
        var order = OrderBuilder.New()
            .WithItem(ProductId.New(), 1, Money.Of(100, "USD"))
            .Build();

        // Act
        order.Place();

        // Assert
        order.DomainEvents.Should().ContainSingle()
            .Which.Should().BeOfType<OrderPlacedEvent>();
    }
}
```

### Value object test

```csharp
public class MoneyTests
{
    [Fact]
    public void Add_WithSameCurrency_ReturnsSummedAmount()
    {
        var a = Money.Of(100, "USD");
        var b = Money.Of(50, "USD");

        var result = a + b;

        result.Amount.Should().Be(150);
        result.Currency.Should().Be("USD");
    }

    [Fact]
    public void Add_WithDifferentCurrencies_ThrowsCurrencyMismatchException()
    {
        var a = Money.Of(100, "USD");
        var b = Money.Of(50, "EUR");

        var act = () => { var _ = a + b; };

        act.Should().Throw<CurrencyMismatchException>();
    }

    [Fact]
    public void Equality_TwoInstancesWithSameValues_AreEqual()
    {
        var a = Money.Of(100, "USD");
        var b = Money.Of(100, "USD");

        a.Should().Be(b);
    }
}
```

### Test data builder (Object Mother pattern)

```csharp
public class OrderBuilder
{
    private OrderId _id = OrderId.New();
    private CustomerId _customerId = CustomerId.New();
    private readonly List<(ProductId, int, Money)> _items = new();
    private OrderStatus? _forcedStatus;

    public static OrderBuilder New() => new();

    public OrderBuilder WithItem(ProductId productId, int quantity, Money price)
    {
        _items.Add((productId, quantity, price));
        return this;
    }

    public OrderBuilder InStatus(OrderStatus status)
    {
        _forcedStatus = status;
        return this;
    }

    public Order Build()
    {
        var order = new Order(_id, _customerId);

        foreach (var (productId, quantity, price) in _items)
            order.AddItem(productId, quantity, price);

        // Use reflection to force a specific status for edge case tests
        if (_forcedStatus.HasValue)
            typeof(Order).GetProperty(nameof(Order.Status))!
                .SetValue(order, _forcedStatus.Value);

        order.ClearDomainEvents(); // start clean for assertion purposes
        return order;
    }
}
```

---

## Important Notes

- Every test method tests **one behavior**. If a test has multiple `// Act` sections, split it.
- The `// Arrange, Act, Assert` comment structure (AAA) is the standard — follow it consistently.
- Test builders prevent test setup from drowning test intent. A test with 20 lines of setup before the act is unreadable — extract the setup into a builder.
- Domain tests never use mocks. If you find yourself mocking something in a domain test, the domain object has an infrastructure dependency it shouldn't have.
