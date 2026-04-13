# Infrastructure / Outbox

## Why This Folder Exists

The Outbox pattern solves the **dual write problem** — the impossibility of atomically writing to a database and publishing to a message broker in a single operation.

Without the Outbox, a service that saves an order and then publishes `OrderPlacedIntegrationEvent` has a race condition: if the process crashes between the DB commit and the broker publish, the event is lost forever. Other services never know the order was placed.

The Outbox eliminates this by writing the event to an `outbox_messages` table **inside the same database transaction** as the business data. A background processor then reads undelivered outbox records and publishes them to the broker. The worst case becomes duplicate delivery (at-least-once), never silent loss.

---

## What Belongs Here

- `OutboxMessage.cs` — the entity that represents a pending event in the outbox table
- `OutboxProcessor.cs` — the background service that reads and publishes outbox records
- `OutboxMessageConfiguration.cs` — EF Core configuration for the outbox table (lives in `Persistence/Configurations/` but referenced here for clarity)

---

## What Does NOT Belong Here

- Integration event class definitions (those live in `Application/IntegrationEvents/`)
- Business logic
- The `IIntegrationEventPublisher` interface (that lives in `Application/Abstractions/`)

---

## OutboxMessage Entity

```csharp
// OutboxMessage.cs
public class OutboxMessage
{
    public Guid Id { get; private set; } = Guid.NewGuid();
    public string EventType { get; private set; }       // fully qualified type name
    public string Payload { get; private set; }         // JSON-serialized event
    public DateTime CreatedAt { get; private set; }
    public DateTime? ProcessedAt { get; private set; }
    public string? Error { get; private set; }
    public int RetryCount { get; private set; }

    private OutboxMessage() { } // EF constructor

    public OutboxMessage(string eventType, string payload)
    {
        EventType = eventType;
        Payload = payload;
        CreatedAt = DateTime.UtcNow;
    }

    public void MarkProcessed() => ProcessedAt = DateTime.UtcNow;

    public void MarkFailed(string error)
    {
        Error = error;
        RetryCount++;
    }
}
```

---

## OutboxProcessor

The background service that polls for unprocessed outbox records and publishes them:

```csharp
// OutboxProcessor.cs
public class OutboxProcessor : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<OutboxProcessor> _logger;
    private static readonly TimeSpan _interval = TimeSpan.FromSeconds(10);

    public OutboxProcessor(
        IServiceScopeFactory scopeFactory,
        ILogger<OutboxProcessor> logger)
    {
        _scopeFactory = scopeFactory;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await ProcessPendingMessagesAsync(stoppingToken);
            await Task.Delay(_interval, stoppingToken);
        }
    }

    private async Task ProcessPendingMessagesAsync(CancellationToken ct)
    {
        using var scope = _scopeFactory.CreateScope();
        var context = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        var publisher = scope.ServiceProvider.GetRequiredService<IPublishEndpoint>();

        var pending = await context.OutboxMessages
            .Where(m => m.ProcessedAt == null && m.RetryCount < 5)
            .OrderBy(m => m.CreatedAt)
            .Take(20)
            .ToListAsync(ct);

        foreach (var message in pending)
        {
            try
            {
                var eventType = Type.GetType(message.EventType)!;
                var @event = JsonSerializer.Deserialize(message.Payload, eventType)!;

                await publisher.Publish(@event, eventType, ct);
                message.MarkProcessed();

                _logger.LogInformation(
                    "Outbox: published {EventType} ({MessageId})",
                    message.EventType, message.Id);
            }
            catch (Exception ex)
            {
                message.MarkFailed(ex.Message);
                _logger.LogError(ex,
                    "Outbox: failed to publish {EventType} ({MessageId})",
                    message.EventType, message.Id);
            }
        }

        await context.SaveChangesAsync(ct);
    }
}
```

---

## Writing to the Outbox

The Outbox is written from within `IIntegrationEventPublisher`'s implementation when using a custom outbox (not MassTransit's built-in):

```csharp
// Custom outbox publisher implementation
public class OutboxIntegrationEventPublisher : IIntegrationEventPublisher
{
    private readonly AppDbContext _context;

    public OutboxIntegrationEventPublisher(AppDbContext context)
        => _context = context;

    public Task PublishAsync<T>(T integrationEvent, CancellationToken ct = default)
        where T : IntegrationEvent
    {
        var message = new OutboxMessage(
            eventType: typeof(T).AssemblyQualifiedName!,
            payload: JsonSerializer.Serialize(integrationEvent));

        _context.OutboxMessages.Add(message);
        // No SaveChanges here — the Unit of Work saves everything atomically
        return Task.CompletedTask;
    }
}
```

This write happens inside the same `SaveChangesAsync` call that saves the aggregate — that is the atomicity guarantee.

---

## Java / Spring Implementation

```java
// OutboxMessage.java — JPA entity
@Entity
@Table(name = "outbox_messages")
public class OutboxMessage {
    @Id
    private UUID id = UUID.randomUUID();
    private String eventType;

    @Column(columnDefinition = "TEXT")
    private String payload;

    private Instant createdAt = Instant.now();
    private Instant processedAt;
    private String error;
    private int retryCount = 0;
}

// OutboxProcessor.java — Spring scheduled task
@Component
public class OutboxProcessor {

    @Scheduled(fixedDelay = 10000)
    @Transactional
    public void processPending() {
        List<OutboxMessage> pending = outboxRepository
            .findTop20ByProcessedAtIsNullAndRetryCountLessThan(5);

        for (OutboxMessage message : pending) {
            try {
                Object event = objectMapper.readValue(
                    message.getPayload(),
                    Class.forName(message.getEventType()));
                streamBridge.send(message.getEventType() + "-out-0", event);
                message.markProcessed();
            } catch (Exception e) {
                message.markFailed(e.getMessage());
            }
        }
    }
}
```

---

## MassTransit Built-In Outbox (Alternative)

If using MassTransit, you can use its built-in Outbox instead of a custom implementation. It handles the write-ahead log internally:

```csharp
x.UsingRabbitMq((context, cfg) =>
{
    cfg.UseInMemoryOutbox(context);   // or cfg.UseEntityFrameworkOutbox<AppDbContext>()
    cfg.ConfigureEndpoints(context);
});
```

`UseEntityFrameworkOutbox<AppDbContext>` is the production-grade option — it uses the same `AppDbContext` to write outbox records within the same transaction as your aggregate.

---

## Outbox Table Definition

```sql
CREATE TABLE outbox_messages (
    id              UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    event_type      NVARCHAR(500)   NOT NULL,
    payload         NVARCHAR(MAX)   NOT NULL,
    created_at      DATETIME2       NOT NULL,
    processed_at    DATETIME2       NULL,
    error           NVARCHAR(1000)  NULL,
    retry_count     INT             NOT NULL DEFAULT 0
);

CREATE INDEX ix_outbox_messages_unprocessed
    ON outbox_messages (created_at)
    WHERE processed_at IS NULL;
```

---

## Important Notes

- The polling interval of the `OutboxProcessor` directly affects event delivery latency. A 10-second interval means events may arrive at consumers up to 10 seconds after the producing transaction commits. Tune based on your SLA.
- Implement a **maximum retry count** (e.g. 5 attempts) and a **dead letter mechanism** for messages that consistently fail to publish. Infinite retries can starve the processor.
- For high-throughput services, consider a CDC (Change Data Capture) approach with Debezium instead of a polling processor. Debezium streams changes from the DB transaction log directly to Kafka without polling latency.
- Periodically clean up processed outbox records. Keep them for a configurable retention window (e.g. 7 days) for debugging, then archive or delete them.
- Consumers of your integration events must be **idempotent** — the at-least-once guarantee means they will occasionally receive duplicates. Use the `EventId` from the `IntegrationEvent` base class as a deduplication key.
