# MyProject.Infrastructure.Tests (v0.3)

## What Changed From v0.2

All v0.2 infrastructure tests still apply (repository, EF configuration, service adapter tests). v0.3 adds tests for the two new infrastructure concerns:

- **Messaging tests** — verify that consumers correctly forward events to MediatR, and that the publisher correctly calls the broker client
- **Outbox tests** — verify that the OutboxProcessor reads pending records and publishes them, handles failures with retry, and marks records as processed

---

## New Test Areas

### Messaging / Consumer Tests

Consumer tests verify the bridge pattern: broker message in → MediatR publish out. Use a mock mediator to assert the right event is forwarded:

```csharp
public class StockDepletedConsumerTests
{
    private readonly IMediator _mediator = Substitute.For<IMediator>();
    private readonly StockDepletedConsumer _consumer;

    public StockDepletedConsumerTests()
        => _consumer = new StockDepletedConsumer(_mediator, NullLogger<StockDepletedConsumer>.Instance);

    [Fact]
    public async Task Consume_ValidMessage_ForwardsToMediator()
    {
        // Arrange
        var message = new InventoryStockDepletedIntegrationEvent(
            ProductId: Guid.NewGuid(),
            RemainingStock: 0);

        var context = Substitute.For<ConsumeContext<InventoryStockDepletedIntegrationEvent>>();
        context.Message.Returns(message);
        context.CancellationToken.Returns(CancellationToken.None);

        // Act
        await _consumer.Consume(context);

        // Assert
        await _mediator.Received(1).Publish(message, Arg.Any<CancellationToken>());
    }
}
```

### Outbox Processor Tests

Outbox processor tests require a real (or containerized) database to verify the read-mark-publish cycle:

```csharp
[Collection("Database")]
public class OutboxProcessorTests : IClassFixture<DatabaseFixture>
{
    private readonly AppDbContext _context;
    private readonly IPublishEndpoint _broker = Substitute.For<IPublishEndpoint>();

    public OutboxProcessorTests(DatabaseFixture fixture)
        => _context = fixture.DbContext;

    [Fact]
    public async Task ProcessPending_WithUnprocessedMessage_PublishesAndMarksProcessed()
    {
        // Arrange
        var message = new OutboxMessage(
            eventType: typeof(OrderPlacedIntegrationEvent).AssemblyQualifiedName!,
            payload: JsonSerializer.Serialize(new OrderPlacedIntegrationEvent(
                Guid.NewGuid(), Guid.NewGuid(), 100m, "USD", [])));

        _context.OutboxMessages.Add(message);
        await _context.SaveChangesAsync();

        var processor = new OutboxProcessor(
            new FakeServiceScopeFactory(_context, _broker),
            NullLogger<OutboxProcessor>.Instance);

        // Act
        await processor.ProcessPendingAsync(CancellationToken.None);

        // Assert
        _context.ChangeTracker.Clear();
        var processed = await _context.OutboxMessages.FindAsync(message.Id);
        processed!.ProcessedAt.Should().NotBeNull();
        await _broker.Received(1).Publish(Arg.Any<object>(), Arg.Any<Type>(), Arg.Any<CancellationToken>());
    }

    [Fact]
    public async Task ProcessPending_WhenBrokerFails_IncrementsRetryCount()
    {
        // Arrange
        _broker.Publish(Arg.Any<object>(), Arg.Any<Type>(), Arg.Any<CancellationToken>())
            .ThrowsAsync(new Exception("Broker unavailable"));

        var message = new OutboxMessage(
            typeof(OrderPlacedIntegrationEvent).AssemblyQualifiedName!,
            JsonSerializer.Serialize(new OrderPlacedIntegrationEvent(
                Guid.NewGuid(), Guid.NewGuid(), 100m, "USD", [])));

        _context.OutboxMessages.Add(message);
        await _context.SaveChangesAsync();

        var processor = new OutboxProcessor(
            new FakeServiceScopeFactory(_context, _broker),
            NullLogger<OutboxProcessor>.Instance);

        // Act
        await processor.ProcessPendingAsync(CancellationToken.None);

        // Assert
        _context.ChangeTracker.Clear();
        var failed = await _context.OutboxMessages.FindAsync(message.Id);
        failed!.RetryCount.Should().Be(1);
        failed.ProcessedAt.Should().BeNull();
        failed.Error.Should().NotBeNullOrEmpty();
    }
}
```

---

## Important Notes

- Consumer tests are **pure unit tests** — no broker, no database, just mock MediatR
- Outbox processor tests are **integration tests** — they need a real database to verify the persistence of the processed/failed state
- The broker (`IPublishEndpoint` / `StreamBridge`) is always mocked in outbox tests — the outbox processor test verifies the read-persist-publish cycle, not the broker behavior
- In Java/Spring, use `@SpringBootTest` with `@MockBean StreamBridge` for outbox processor tests, and plain Mockito for consumer unit tests
