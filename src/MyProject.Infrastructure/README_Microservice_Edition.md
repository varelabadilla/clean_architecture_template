# MyProject.Infrastructure (v0.3)

## What Changed From v0.2

The Infrastructure layer gains two new top-level folders:

- `Messaging/` — MassTransit consumers and publishers that connect the service to the message broker
- `Outbox/` — the Outbox pattern implementation: the processor that reads undelivered events and publishes them to the broker

Everything else — `Persistence/`, `Services/`, `Integrations/`, `Identity/`, `Caching/`, `Events/` — is unchanged from v0.2.

---

## Updated Folder Overview

```plaintext
MyProject.Infrastructure/
├─ Persistence/           ← unchanged from v0.2
│  ├─ Configurations/     ← + OutboxMessageConfiguration.cs (new)
│  ├─ Migrations/
│  ├─ Repositories/
│  └─ AppDbContext.cs
├─ Events/                ← unchanged from v0.2
├─ Messaging/             ← NEW
│  ├─ Consumers/          ← MassTransit IConsumer<T> implementations
│  └─ Publishers/         ← IIntegrationEventPublisher implementation
├─ Outbox/                ← NEW
│  ├─ OutboxMessage.cs
│  ├─ OutboxProcessor.cs
│  └─ IOutboxRepository.cs (optional)
├─ Services/              ← unchanged from v0.2
├─ Integrations/          ← unchanged from v0.2
├─ Identity/              ← unchanged from v0.2
├─ Caching/               ← unchanged from v0.2
└─ DependencyInjection.cs ← updated: registers messaging + outbox
```

---

## Updated DependencyInjection.cs

```csharp
public static IServiceCollection AddInfrastructureServices(
    this IServiceCollection services,
    IConfiguration configuration)
{
    // --- Persistence (unchanged) ---
    services.AddDbContext<AppDbContext>(options =>
        options.UseSqlServer(configuration.GetConnectionString("DefaultConnection")));
    services.AddScoped<IOrderRepository, OrderRepository>();
    services.AddScoped<IUnitOfWork>(sp => sp.GetRequiredService<AppDbContext>());
    services.AddScoped<IDomainEventDispatcher, DomainEventDispatcher>();

    // --- Messaging (new) ---
    services.AddMassTransit(x =>
    {
        // Register all consumers in this assembly automatically
        x.AddConsumers(Assembly.GetExecutingAssembly());

        x.UsingRabbitMq((context, cfg) =>
        {
            cfg.Host(configuration.GetConnectionString("RabbitMq"));

            // Outbox: guarantees at-least-once delivery
            cfg.UseMessageRetry(r => r.Intervals(500, 1000, 3000));
            cfg.UseInMemoryOutbox(context); // MassTransit built-in outbox

            cfg.ConfigureEndpoints(context);
        });
    });

    services.AddScoped<IIntegrationEventPublisher, MassTransitIntegrationEventPublisher>();

    // --- Outbox processor (new) ---
    services.AddHostedService<OutboxProcessor>();

    // --- Services (unchanged) ---
    services.AddScoped<IEmailService, SendGridEmailService>();
    services.AddSingleton<ICacheService, RedisCacheService>();
    services.AddScoped<ICurrentUserContext, HttpCurrentUserContext>();
    services.AddSingleton<IDateTimeProvider, SystemDateTimeProvider>();

    return services;
}
```

---

## Important Notes

- The Messaging and Outbox folders are the only additions to Infrastructure in v0.3. The dependency rule is unchanged — Infrastructure still implements contracts from Application and Domain, never the other way around.
- `IIntegrationEventPublisher` is defined in `Application/Abstractions/` and implemented here in `Messaging/Publishers/`. Application never references MassTransit directly.
- In Java/Spring, messaging configuration uses `@EnableBinding` or `@Configuration` beans for Spring Cloud Stream, with `@StreamListener` consumer adapters in `Messaging/Consumers/`.
