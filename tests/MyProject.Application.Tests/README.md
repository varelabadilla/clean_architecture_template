# MyProject.Application.Tests

## Why This Project Exists

Application tests validate **use case behavior** — that handlers correctly orchestrate domain operations, that validators reject invalid input, and that behaviors in the pipeline do what they are supposed to do.

These are unit tests. Repositories and external services are replaced with fakes or mocks so tests remain fast and isolated from infrastructure.

---

## What to Test Here

- **Command handlers:** Given valid input and a repository that returns a known aggregate, does the handler produce the expected outcome?
- **Query handlers:** Does the handler call the repository with the right parameters and map the result correctly?
- **Validators:** Does the validator reject invalid input? Does it pass valid input?
- **Behaviors:** Does `ValidationBehavior` short-circuit on validation failure? Does `LoggingBehavior` log at the right level?
- **Event handlers:** Does the domain event handler call the right service with the right data?

---

## What NOT to Test Here

- Domain invariants (those belong in Domain.Tests)
- Repository SQL correctness (that belongs in Infrastructure.Tests)
- HTTP routing or response codes (that belongs in Api.Tests)

---

## Recommended Project Structure

```
MyProject.Application.Tests/
├─ Features/
│  └─ Orders/
│     ├─ Commands/
│     │  └─ PlaceOrderCommandHandlerTests.cs
│     ├─ Queries/
│     │  └─ GetOrderByIdQueryHandlerTests.cs
│     └─ Validators/
│        └─ PlaceOrderCommandValidatorTests.cs
├─ Behaviors/
│  ├─ ValidationBehaviorTests.cs
│  └─ LoggingBehaviorTests.cs
└─ Common/
   └─ Fakes/
      ├─ FakeOrderRepository.cs
      ├─ FakeEmailService.cs
      └─ FakeDateTimeProvider.cs
```

---

## Naming Conventions

Same as Domain.Tests:
- Test class: `{HandlerOrClass}Tests`
- Method: `{Method}_{Condition}_{ExpectedOutcome}`

---

## Test Examples

### Command handler test
```csharp
public class PlaceOrderCommandHandlerTests
{
    private readonly IOrderRepository _orders;
    private readonly IProductRepository _products;
    private readonly PlaceOrderCommandHandler _handler;

    public PlaceOrderCommandHandlerTests()
    {
        _orders = Substitute.For<IOrderRepository>();
        _products = Substitute.For<IProductRepository>();
        _handler = new PlaceOrderCommandHandler(_orders, _products);
    }

    [Fact]
    public async Task Handle_WithValidCommand_ReturnsSuccessWithOrderId()
    {
        // Arrange
        var productId = ProductId.New();
        var product = ProductBuilder.New().WithId(productId).WithPrice(100, "USD").Build();
        _products.GetByIdAsync(productId, default).Returns(product);

        var command = new PlaceOrderCommand(
            CustomerId: Guid.NewGuid(),
            Items: [new OrderItemRequest(productId.Value, 2)],
            ShippingAddress: AddressRequestFaker.Valid());

        // Act
        var result = await _handler.Handle(command, CancellationToken.None);

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.Should().NotBeEmpty();
        await _orders.Received(1).AddAsync(Arg.Any<Order>(), Arg.Any<CancellationToken>());
    }

    [Fact]
    public async Task Handle_WhenProductNotFound_ReturnsNotFoundError()
    {
        // Arrange
        _products.GetByIdAsync(Arg.Any<ProductId>(), default).Returns((Product?)null);

        var command = new PlaceOrderCommand(
            CustomerId: Guid.NewGuid(),
            Items: [new OrderItemRequest(Guid.NewGuid(), 1)],
            ShippingAddress: AddressRequestFaker.Valid());

        // Act
        var result = await _handler.Handle(command, CancellationToken.None);

        // Assert
        result.IsFailure.Should().BeTrue();
        result.Error.Type.Should().Be(ErrorType.NotFound);
    }
}
```

### Validator test
```csharp
public class PlaceOrderCommandValidatorTests
{
    private readonly PlaceOrderCommandValidator _validator = new();

    [Fact]
    public void Validate_WithValidCommand_PassesValidation()
    {
        var command = PlaceOrderCommandFaker.Valid();
        var result = _validator.Validate(command);
        result.IsValid.Should().BeTrue();
    }

    [Fact]
    public void Validate_WithEmptyItems_FailsValidation()
    {
        var command = PlaceOrderCommandFaker.Valid() with { Items = [] };
        var result = _validator.Validate(command);

        result.IsValid.Should().BeFalse();
        result.Errors.Should().Contain(e =>
            e.PropertyName == nameof(PlaceOrderCommand.Items));
    }

    [Fact]
    public void Validate_WithZeroQuantity_FailsValidation()
    {
        var command = PlaceOrderCommandFaker.Valid() with
        {
            Items = [new OrderItemRequest(Guid.NewGuid(), 0)]
        };
        var result = _validator.Validate(command);

        result.IsValid.Should().BeFalse();
        result.Errors.Should().Contain(e => e.PropertyName.Contains("Quantity"));
    }
}
```

### Fake repository
```csharp
// Fakes are preferred over mocks for stateful repositories
public class FakeOrderRepository : IOrderRepository
{
    private readonly List<Order> _store = new();

    public Task<Order?> GetByIdAsync(OrderId id, CancellationToken ct = default)
        => Task.FromResult(_store.FirstOrDefault(o => o.Id == id));

    public Task<IReadOnlyList<Order>> FindAsync(ISpecification<Order> spec, CancellationToken ct = default)
        => Task.FromResult<IReadOnlyList<Order>>(
            _store.Where(spec.IsSatisfiedBy).ToList());

    public Task<bool> ExistsAsync(OrderId id, CancellationToken ct = default)
        => Task.FromResult(_store.Any(o => o.Id == id));

    public Task AddAsync(Order order, CancellationToken ct = default)
    {
        _store.Add(order);
        return Task.CompletedTask;
    }

    public void Update(Order order) { } // in-memory — already mutated
    public void Delete(Order order) => _store.Remove(order);
}
```

---

## Important Notes

- Prefer **fakes over mocks** for repositories and stateful services. Fakes have real behavior; mocks only verify calls. A fake `FakeOrderRepository` that stores entities in a list is more realistic and produces better test failures.
- Use mocks (NSubstitute/Mockito) for **fire-and-forget services** like `IEmailService` where you only need to verify the call was made.
- Test one scenario per test method. Parameterized tests (`[Theory]` in xUnit) are acceptable for boundary conditions (empty string, null, zero, max value).
