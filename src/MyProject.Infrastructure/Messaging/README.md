# Infrastructure / Messaging

## Why This Folder Exists

This folder is the **bridge between the application and the message broker**. It contains two things:

- **Publishers** — the concrete implementation of `IIntegrationEventPublisher` that hands messages to MassTransit or Spring Cloud Stream
- **Consumers** — the broker-specific entry points that receive messages from other services and forward them into the application via MediatR

All broker-specific code lives here and only here. The Application layer never knows that MassTransit, RabbitMQ, or Kafka exists.

---

## Folder Structure

```plaintext
Messaging/
├─ Publishers/
│  └─ MassTransitIntegrationEventPublisher.cs    // implements IIntegrationEventPublisher
└─ Consumers/
   ├─ StockDepletedConsumer.cs                   // consumes from InventoryService
   └─ PaymentRefundedConsumer.cs                 // consumes from PaymentService
```

---

## What Belongs Here

- `IIntegrationEventPublisher` implementation (publisher)
- MassTransit `IConsumer<T>` classes (consumers) — one per consumed integration event type
- Consumer retry and error handling configuration specific to each consumer
- Dead letter queue configuration

---

## What Does NOT Belong Here

- `IIntegrationEventPublisher` interface (that lives in `Application/Abstractions/`)
- Integration event class definitions (those live in `Application/IntegrationEvents/`)
- Application handler logic (that lives in `Application/Features/{Feature}/EventHandlers/`)
- Business logic of any kind

---

## Naming Conventions

| Type | Convention | Example |
| --- | --- | --- |
| Publisher implementation | Technology + `IntegrationEventPublisher` | `MassTransitIntegrationEventPublisher` |
| Consumer class | EventName + `Consumer` | `StockDepletedConsumer`, `PaymentRefundedConsumer` |

---

## Publishers/

### MassTransitIntegrationEventPublisher (.NET)

```csharp
public class MassTransitIntegrationEventPublisher : IIntegrationEventPublisher
{
    private readonly IPublishEndpoint _publishEndpoint;

    public MassTransitIntegrationEventPublisher(IPublishEndpoint publishEndpoint)
        => _publishEndpoint = publishEndpoint;

    public async Task PublishAsync<T>(
        T integrationEvent,
        CancellationToken cancellationToken = default)
        where T : IntegrationEvent
    {
        await _publishEndpoint.Publish(integrationEvent, cancellationToken);
    }
}
```

### Spring Cloud Stream Publisher (Java)

```java
@Service
public class SpringCloudStreamIntegrationEventPublisher
    implements IntegrationEventPublisher {

    private final StreamBridge streamBridge;

    public SpringCloudStreamIntegrationEventPublisher(StreamBridge streamBridge) {
        this.streamBridge = streamBridge;
    }

    @Override
    public <T extends IntegrationEvent> void publish(T integrationEvent) {
        String bindingName = integrationEvent.getClass().getSimpleName() + "-out-0";
        streamBridge.send(bindingName, integrationEvent);
    }
}
```

---

## Consumers/

Consumers are the entry points for messages arriving from the broker. Their responsibility is narrow: deserialize the message and forward it to MediatR so the Application layer handles the business reaction.

### MassTransit Consumer (.NET)

```csharp
// StockDepletedConsumer.cs
public class StockDepletedConsumer
    : IConsumer<InventoryStockDepletedIntegrationEvent>
{
    private readonly IMediator _mediator;
    private readonly ILogger<StockDepletedConsumer> _logger;

    public StockDepletedConsumer(IMediator mediator, ILogger<StockDepletedConsumer> logger)
    {
        _mediator = mediator;
        _logger = logger;
    }

    public async Task Consume(ConsumeContext<InventoryStockDepletedIntegrationEvent> context)
    {
        _logger.LogInformation(
            "Received StockDepleted event for product {ProductId}",
            context.Message.ProductId);

        // Forward to MediatR — Application layer handles the reaction
        await _mediator.Publish(context.Message, context.CancellationToken);
    }
}
```

### Spring Cloud Stream Consumer (Java)

```java
@Component
public class StockDepletedConsumer {

    private final ApplicationEventPublisher eventPublisher;

    public StockDepletedConsumer(ApplicationEventPublisher eventPublisher) {
        this.eventPublisher = eventPublisher;
    }

    @Bean
    public Consumer<InventoryStockDepletedIntegrationEvent> stockDepleted() {
        return event -> {
            log.info("Received StockDepleted event for product {}", event.getProductId());
            // Forward to Spring application events — service layer handles the reaction
            eventPublisher.publishEvent(event);
        };
    }
}
```

---

## Queue and Topic Naming Conventions

Use a consistent naming scheme across all services to avoid collisions and confusion on the broker:

**.NET / RabbitMQ with MassTransit:**
MassTransit auto-generates queue names from the consumer type name by default. To make them explicit:

```csharp
cfg.ReceiveEndpoint("order-service.stock-depleted", e =>
{
    e.ConfigureConsumer<StockDepletedConsumer>(context);
});
```

Convention: `{consuming-service}.{event-name-kebab-case}`

**Java / Spring Cloud Stream / Kafka:**

```yaml
# application.yml
spring:
  cloud:
    stream:
      bindings:
        stockDepleted-in-0:
          destination: inventory.stock-depleted   # topic name
          group: order-service                    # consumer group
        orderPlaced-out-0:
          destination: order.order-placed
```

Convention for topics: `{publishing-service}.{event-name-kebab-case}`
Convention for consumer groups: `{consuming-service}`

---

## Error Handling and Retry

**MassTransit (.NET):**

```csharp
cfg.ReceiveEndpoint("order-service.stock-depleted", e =>
{
    e.UseMessageRetry(r => r.Intervals(500, 1000, 5000, 15000));
    e.UseDeadLetterQueue("order-service.stock-depleted.dlq");
    e.ConfigureConsumer<StockDepletedConsumer>(context);
});
```

**Spring Cloud Stream (Java):**

```yaml
spring:
  cloud:
    stream:
      rabbit:
        bindings:
          stockDepleted-in-0:
            consumer:
              auto-bind-dlq: true
              republish-to-dlq: true
              max-attempts: 3
              back-off-initial-interval: 500
```

---

## Important Notes

- Consumers must be **idempotent** — the broker guarantees at-least-once delivery, which means a consumer may receive the same message more than once. Use the event's `EventId` as an idempotency key, checked against a processed events log or the Outbox.
- Never put business logic in consumers. A consumer that grows beyond deserialize + forward is a sign that logic has leaked from the Application layer.
- In production, always configure a **Dead Letter Queue (DLQ)**. Messages that fail after all retries go there for manual inspection instead of being lost.
- Monitor consumer lag (the number of unprocessed messages in a queue/topic). Sudden increases indicate a processing bottleneck or a downstream failure.
