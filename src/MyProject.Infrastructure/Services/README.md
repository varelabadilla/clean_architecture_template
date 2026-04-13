# Infrastructure / Services

## Why This Folder Exists

This folder contains the **concrete implementations of the service interfaces** defined in `Application/Abstractions/`. When the Application layer declares `IEmailService` or `IDateTimeProvider`, this folder provides the real implementations that talk to SendGrid, the system clock, or any other concrete dependency.

---

## What Belongs Here

- `IEmailService` implementation (e.g., `SendGridEmailService`, `SmtpEmailService`)
- `IDateTimeProvider` implementation (`SystemDateTimeProvider`)
- `ICurrentUserContext` implementation (`HttpCurrentUserContext`)
- Any application abstraction implementation that doesn't belong in `Integrations/` or `Caching/`

---

## What Does NOT Belong Here

- Service interfaces (those live in `Application/Abstractions/`)
- Repository implementations (those live in `Persistence/Repositories/`)
- External API-specific clients (those live in `Integrations/`)
- Business logic

---

## Recommended Naming Conventions

| Type | Convention | Example |
| --- | --- | --- |
| Service implementation | Provider/Technology + InterfaceName without `I` | `SendGridEmailService`, `SystemDateTimeProvider` |
| Fallback/stub impl | `Null` + name or `NoOp` + name | `NullEmailService` (for dev/test) |

---

## Common Implementations

### SystemDateTimeProvider

```csharp
public class SystemDateTimeProvider : IDateTimeProvider
{
    public DateTime UtcNow => DateTime.UtcNow;
    public DateOnly Today => DateOnly.FromDateTime(DateTime.UtcNow);
}
```

### HttpCurrentUserContext

```csharp
public class HttpCurrentUserContext : ICurrentUserContext
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    public HttpCurrentUserContext(IHttpContextAccessor httpContextAccessor)
        => _httpContextAccessor = httpContextAccessor;

    public Guid UserId =>
        Guid.Parse(_httpContextAccessor.HttpContext?.User
            .FindFirstValue(ClaimTypes.NameIdentifier) ?? Guid.Empty.ToString());

    public string Email =>
        _httpContextAccessor.HttpContext?.User
            .FindFirstValue(ClaimTypes.Email) ?? string.Empty;

    public IReadOnlyList<string> Roles =>
        _httpContextAccessor.HttpContext?.User
            .FindAll(ClaimTypes.Role)
            .Select(c => c.Value)
            .ToList() ?? new List<string>();

    public bool IsAuthenticated =>
        _httpContextAccessor.HttpContext?.User.Identity?.IsAuthenticated ?? false;
}
```

### NullEmailService (for development)

```csharp
// Logs emails instead of sending them — useful in local development
public class NullEmailService : IEmailService
{
    private readonly ILogger<NullEmailService> _logger;

    public NullEmailService(ILogger<NullEmailService> logger) => _logger = logger;

    public Task SendAsync(string to, string subject, string body,
        bool isHtml = true, CancellationToken cancellationToken = default)
    {
        _logger.LogInformation(
            "[NULL EMAIL] To: {To} | Subject: {Subject}", to, subject);
        return Task.CompletedTask;
    }
}
```

---

## Important Notes

- Register the appropriate implementation based on environment in `DependencyInjection.cs`:

```csharp
if (environment.IsDevelopment())
    services.AddScoped<IEmailService, NullEmailService>();
else
    services.AddScoped<IEmailService, SendGridEmailService>();
```

- Keep implementations **thin adapters**. If a service implementation grows complex, it likely contains logic that belongs in the Application layer.
- In Java/Spring, these are `@Service` beans that implement the domain/application interfaces, registered automatically via component scanning.
