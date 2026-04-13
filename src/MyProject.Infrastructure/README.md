# MyProject.Infrastructure

## Why This Project Exists

The Infrastructure layer is where **technical concerns live**. It implements every interface defined in the Domain and Application layers using concrete frameworks, libraries, and external services.

This is the only layer that is allowed — and expected — to be coupled to specific technologies: Entity Framework Core or Dapper for persistence, SendGrid or SMTP for email, Redis for caching, RabbitMQ or Azure Service Bus for messaging. That coupling is intentional and healthy here, because it is isolated. The rest of the application never knows which concrete technology is used.

**The key principle:** Infrastructure implements; it does not define. Every contract it fulfills was defined somewhere else — in Domain or Application.

---

## What Belongs Here

- Database context and ORM configuration
- Repository implementations
- External service clients and adapters (email, storage, payment, SMS)
- Identity and authentication infrastructure
- Caching implementations
- Message broker adapters
- Domain event dispatcher implementation
- Background job infrastructure

---

## What Does NOT Belong Here

- Business rules (those live in Domain)
- Use case orchestration (that lives in Application)
- HTTP controllers or filters (those live in Api)
- Interfaces or contracts (those live in Domain or Application)

---

## Dependency Rule

```plaintext
MyProject.Infrastructure → MyProject.Application, MyProject.Domain, MyProject.SharedKernel
```

Infrastructure may reference Application to implement its abstractions. Application must never reference Infrastructure.

---

## Recommended Naming Conventions

| Type | Convention | Example |
| --- | --- | --- |
| Repository impl | AggregateName + `Repository` | `OrderRepository`, `CustomerRepository` |
| Service impl | Technology + ServiceName | `SendGridEmailService`, `RedisCache Service` |
| EF configuration | EntityName + `Configuration` | `OrderConfiguration`, `OrderItemConfiguration` |
| Integration adapter | Provider + concept | `StripePaymentAdapter`, `TwilioSmsService` |
| DI extension | `DependencyInjection.cs` | Fixed name per layer |

---

## DependencyInjection.cs

This file is the Infrastructure layer's registration entry point. It is called from the API's Composition Root:

```csharp
public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructureServices(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        // Persistence
        services.AddDbContext<AppDbContext>(options =>
            options.UseSqlServer(configuration.GetConnectionString("DefaultConnection")));

        services.AddScoped<IOrderRepository, OrderRepository>();
        services.AddScoped<ICustomerRepository, CustomerRepository>();
        services.AddScoped<IUnitOfWork>(sp => sp.GetRequiredService<AppDbContext>());

        // Domain Events
        services.AddScoped<IDomainEventDispatcher, DomainEventDispatcher>();

        // Services
        services.AddScoped<IEmailService, SendGridEmailService>();
        services.AddSingleton<ICacheService, RedisCacheService>();
        services.AddScoped<ICurrentUserContext, HttpCurrentUserContext>();
        services.AddSingleton<IDateTimeProvider, SystemDateTimeProvider>();

        // Caching
        services.AddStackExchangeRedisCache(options =>
            options.Configuration = configuration.GetConnectionString("Redis"));

        return services;
    }
}
```

---

## Folder Overview

| Folder | Contents |
| --- | --- |
| `Persistence/` | DbContext, EF configurations, migrations, repository implementations |
| `Events/` | `DomainEventDispatcher` implementation |
| `Services/` | Concrete implementations of Application abstractions |
| `Integrations/` | External API clients and adapters |
| `Identity/` | Authentication and authorization infrastructure |
| `Caching/` | Cache service implementation and configuration |

---

## Java/Spring Equivalent

In Spring Boot, Infrastructure is typically organized as:

```plaintext
infrastructure/
├─ persistence/
│  ├─ jpa/           // JPA repository implementations
│  ├─ entity/        // JPA entity mappings (@Entity classes)
│  └─ config/        // DataSource, JPA configuration
├─ messaging/        // RabbitMQ / Kafka adapters
├─ email/            // SMTP / SendGrid adapters
├─ cache/            // Redis / Caffeine implementations
└─ security/         // Spring Security configuration
```

`@Configuration` classes within each subfolder serve the same role as `DependencyInjection.cs`.

---

## Important Notes

- Infrastructure classes should implement interfaces from other layers — they should not define their own contracts. If you find yourself writing a `public interface` in Infrastructure, it almost certainly belongs in Application or Domain.
- Prefer composition over inheritance in adapters. An `EmailService` wrapping a `SendGridClient` is better than an `EmailService` extending some framework-provided base class.
- Infrastructure tests are integration tests — they run against real (or containerized) dependencies like a test database or Redis instance.
