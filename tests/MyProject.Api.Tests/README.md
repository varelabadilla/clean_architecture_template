# MyProject.Api.Tests

## Why This Project Exists

Api tests are **end-to-end integration tests** at the HTTP level. They spin up the full application stack — in-process — and make real HTTP requests against it, validating that the entire pipeline from route resolution to database persistence works correctly together.

These tests catch problems that unit tests cannot: incorrect route configuration, missing middleware, broken authentication, wrong HTTP status codes, and serialization issues.

---

## What to Test Here

- **HTTP contracts:** Does `POST /api/orders` return `201 Created` with the expected body?
- **Authentication:** Does an unauthenticated request to a protected endpoint return `401`?
- **Authorization:** Does a request from an unauthorized user return `403`?
- **Validation errors:** Does a request with invalid input return `422` with structured error details?
- **Middleware behavior:** Does `GlobalExceptionMiddleware` return the correct error shape?
- **Full happy path:** Does a complete create → read → update flow work end-to-end?

---

## What NOT to Test Here

- Business logic (Domain.Tests)
- Handler internals (Application.Tests)
- Repository SQL (Infrastructure.Tests)

---

## Recommended Project Structure

```plaintext
MyProject.Api.Tests/
├─ Controllers/
│  ├─ OrdersControllerTests.cs
│  └─ CustomersControllerTests.cs
├─ Middleware/
│  └─ GlobalExceptionMiddlewareTests.cs
├─ Common/
│  ├─ WebAppFactory.cs      // custom WebApplicationFactory
│  ├─ AuthHelper.cs         // JWT token generation for tests
│  └─ ApiTestBase.cs        // base class with HTTP client helpers
```

---

## WebApplicationFactory Setup

```csharp
// Common/WebAppFactory.cs
public class WebAppFactory : WebApplicationFactory<Program>, IAsyncLifetime
{
    private readonly MsSqlContainer _db = new MsSqlBuilder().Build();

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // Replace real DB with test container
            var descriptor = services.SingleOrDefault(d =>
                d.ServiceType == typeof(DbContextOptions<AppDbContext>));
            if (descriptor is not null)
                services.Remove(descriptor);

            services.AddDbContext<AppDbContext>(options =>
                options.UseSqlServer(_db.GetConnectionString()));

            // Replace email service with no-op
            services.AddScoped<IEmailService, NullEmailService>();
        });
    }

    public async Task InitializeAsync()
    {
        await _db.StartAsync();
        using var scope = Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await db.Database.MigrateAsync();
    }

    public new async Task DisposeAsync() => await _db.DisposeAsync();
}
```

---

## Controller Test Example

```csharp
[Collection("Api")]
public class OrdersControllerTests : IClassFixture<WebAppFactory>
{
    private readonly HttpClient _client;

    public OrdersControllerTests(WebAppFactory factory)
    {
        _client = factory.CreateClient();
        _client.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", AuthHelper.GenerateToken());
    }

    [Fact]
    public async Task PlaceOrder_WithValidRequest_Returns201Created()
    {
        // Arrange
        var request = new PlaceOrderRequest(
            CustomerId: Guid.NewGuid(),
            Items: [new OrderItemRequest(Guid.NewGuid(), 2)],
            ShippingAddress: new AddressRequest("Main St", "Springfield", "US", "12345"));

        // Act
        var response = await _client.PostAsJsonAsync("api/orders", request);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Created);
        var orderId = await response.Content.ReadFromJsonAsync<Guid>();
        orderId.Should().NotBeEmpty();
    }

    [Fact]
    public async Task PlaceOrder_WithoutAuthentication_Returns401()
    {
        var client = new HttpClient { BaseAddress = _client.BaseAddress };
        var response = await client.PostAsJsonAsync("api/orders", new { });

        response.StatusCode.Should().Be(HttpStatusCode.Unauthorized);
    }

    [Fact]
    public async Task PlaceOrder_WithEmptyItems_Returns422WithValidationErrors()
    {
        // Arrange
        var request = new PlaceOrderRequest(
            CustomerId: Guid.NewGuid(),
            Items: [],
            ShippingAddress: new AddressRequest("Main St", "Springfield", "US", "12345"));

        // Act
        var response = await _client.PostAsJsonAsync("api/orders", request);
        var error = await response.Content.ReadFromJsonAsync<ErrorResponse>();

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.UnprocessableEntity);
        error!.Errors.Should().ContainKey("Items");
    }

    [Fact]
    public async Task GetOrder_WhenNotFound_Returns404()
    {
        var response = await _client.GetAsync($"api/orders/{Guid.NewGuid()}");
        response.StatusCode.Should().Be(HttpStatusCode.NotFound);
    }
}
```

---

## Important Notes

- Api tests are the slowest and most expensive — keep them focused on **HTTP contract correctness**, not business logic coverage. Business logic coverage belongs in Domain and Application tests.
- Use `WebApplicationFactory<Program>` (built into ASP.NET Core test infrastructure) — it starts the full pipeline in-process without a real network.
- Replace infrastructure services that cause side effects (email, payment, SMS) with no-op implementations in `ConfigureWebHost`.
- `Program.cs` must use `public partial class Program {}` or have `InternalsVisibleTo` configured for `WebApplicationFactory<Program>` to work correctly.
- In Java/Spring, use `@SpringBootTest(webEnvironment = RANDOM_PORT)` with `TestRestTemplate` or MockMvc for the same effect.
