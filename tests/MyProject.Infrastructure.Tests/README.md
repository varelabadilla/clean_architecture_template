# MyProject.Infrastructure.Tests

## Why This Project Exists

Infrastructure tests are **integration tests** — they run against real or containerized dependencies (a real SQL Server or PostgreSQL instance, a real Redis instance) to validate that the infrastructure implementations work correctly.

These tests catch problems that unit tests cannot: incorrect EF Core configurations, broken migrations, SQL queries that return wrong data, and external service adapter failures.

---

## What to Test Here

- **Repository implementations:** Can the repository persist an aggregate and retrieve it with the correct data and relationships?
- **EF Core configurations:** Are value objects mapped correctly? Are strongly-typed IDs persisted and loaded without corruption?
- **Specification evaluation:** Does `SpecificationEvaluator` translate specifications into correct SQL?
- **External service adapters:** Does the adapter call the external API correctly and translate the response? (Use mocked HTTP or a sandbox environment)
- **Domain event dispatcher:** Does the dispatcher call MediatR publish for each event?

---

## What NOT to Test Here

- Business logic (Domain.Tests)
- Handler orchestration (Application.Tests)
- HTTP routing (Api.Tests)

---

## Recommended Project Structure

```
MyProject.Infrastructure.Tests/
├─ Persistence/
│  ├─ Repositories/
│  │  ├─ OrderRepositoryTests.cs
│  │  └─ CustomerRepositoryTests.cs
│  └─ Configurations/
│     └─ OrderConfigurationTests.cs
├─ Services/
│  └─ SendGridEmailServiceTests.cs
├─ Common/
│  ├─ DatabaseFixture.cs       // shared test DB setup
│  └─ IntegrationTestBase.cs   // base class for DB tests
└─ docker-compose.tests.yml    // or use Testcontainers in code
```

---

## Database Setup with Testcontainers

```csharp
// Common/DatabaseFixture.cs
public class DatabaseFixture : IAsyncLifetime
{
    private readonly MsSqlContainer _container = new MsSqlBuilder()
        .WithImage("mcr.microsoft.com/mssql/server:2022-latest")
        .Build();

    public AppDbContext DbContext { get; private set; } = null!;

    public async Task InitializeAsync()
    {
        await _container.StartAsync();

        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlServer(_container.GetConnectionString())
            .Options;

        DbContext = new AppDbContext(
            options,
            new NoOpDomainEventDispatcher(),
            new FakeCurrentUserContext(),
            new FakeDateTimeProvider());

        await DbContext.Database.MigrateAsync();
    }

    public async Task DisposeAsync()
    {
        await DbContext.DisposeAsync();
        await _container.DisposeAsync();
    }
}
```

---

## Repository Integration Test Example

```csharp
[Collection("Database")]
public class OrderRepositoryTests : IClassFixture<DatabaseFixture>
{
    private readonly AppDbContext _context;
    private readonly OrderRepository _repository;

    public OrderRepositoryTests(DatabaseFixture fixture)
    {
        _context = fixture.DbContext;
        _repository = new OrderRepository(_context);
    }

    [Fact]
    public async Task AddAsync_ThenGetById_ReturnsPersistedOrder()
    {
        // Arrange
        var order = OrderBuilder.New()
            .WithItem(ProductId.New(), 2, Money.Of(50, "USD"))
            .Build();

        // Act
        await _repository.AddAsync(order);
        await _context.SaveChangesAsync();

        _context.ChangeTracker.Clear(); // force reload from DB

        var loaded = await _repository.GetByIdAsync(order.Id);

        // Assert
        loaded.Should().NotBeNull();
        loaded!.Id.Should().Be(order.Id);
        loaded.Items.Should().HaveCount(1);
        loaded.TotalAmount.Amount.Should().Be(100);
        loaded.TotalAmount.Currency.Should().Be("USD");
    }

    [Fact]
    public async Task FindAsync_WithSpecification_ReturnsMatchingOrders()
    {
        // Arrange — seed two orders
        var placed = OrderBuilder.New()
            .WithItem(ProductId.New(), 1, Money.Of(50, "USD"))
            .InStatus(OrderStatus.Placed)
            .Build();
        var pending = OrderBuilder.New()
            .WithItem(ProductId.New(), 1, Money.Of(50, "USD"))
            .Build();

        await _repository.AddAsync(placed);
        await _repository.AddAsync(pending);
        await _context.SaveChangesAsync();

        // Act
        var results = await _repository.FindAsync(new PlacedOrdersSpecification());

        // Assert
        results.Should().Contain(o => o.Id == placed.Id);
        results.Should().NotContain(o => o.Id == pending.Id);
    }
}
```

---

## Important Notes

- **Use Testcontainers**, not EF InMemory. The InMemory provider does not support transactions, foreign keys, unique constraints, or computed columns — all of which are real concerns in production.
- Clear the `ChangeTracker` before assertions that need to verify persistence (`_context.ChangeTracker.Clear()`). Without this, EF returns the cached in-memory object instead of querying the database.
- Each test should clean up after itself or run against a fresh database. Use transactions that roll back after each test, or run each test class against a fresh container.
- In Java/Spring, use `@SpringBootTest` with Testcontainers for the database and `@Transactional` on test methods to roll back after each test.
