# Application / IntegrationEvents

## Why This Folder Exists

Integration events are the **public API of this service on the message bus** — the facts this service announces to the rest of the ecosystem when something meaningful happens. `OrderPlacedIntegrationEvent`, `OrderCancelledIntegrationEvent`, `CustomerRegisteredIntegrationEvent` are examples.

This folder defines those contracts. Placing them in the Application layer is deliberate: the Application layer knows *what* this service needs to communicate to the outside world. The *how* (MassTransit, Kafka, RabbitMQ) belongs in Infrastructure.

---

## What Belongs Here

- A base `IntegrationEvent` class with common metadata
- All integration event classes this service publishes
- No classes for events consumed *from* other services — those are copied locally into `Features/{Feature}/EventHandlers/` as consumed event contracts

---

## What Does NOT Belong Here

- Domain events (those live in `Domain/Events/`)
- Event handler implementations (those live in `Features/{Feature}/EventHandlers/`)
- MassTransit, Kafka, or RabbitMQ types — this folder has zero infrastructure dependencies
- Integration events from other services — copy them locally, never reference another service's project

---

## Naming Conventions

| Type | Convention | Example |
| --- | --- | --- |
| Base class | `IntegrationEvent` | Fixed name |
| Published event | Past tense noun phrase + `IntegrationEvent` | `OrderPlacedIntegrationEvent` |
| File name | Same as class name | `OrderPlacedIntegrationEvent.cs` |

---

## Base Class

```csharp
// IntegrationEvent.cs
public abstract record IntegrationEvent
{
    public Guid EventId { get; } = Guid.NewGuid();
    public DateTime OccurredOn { get; } = DateTime.UtcNow;
    public string EventType => GetType().Name;
}
```

```java
// Java
public abstract class IntegrationEvent {
    private final UUID eventId = UUID.randomUUID();
    private final Instant occurredOn = Instant.now();

    public UUID getEventId() { return eventId; }
    public Instant getOccurredOn() { return occurredOn; }
    public String getEventType() { return getClass().getSimpleName(); }
}
```

---

## Integration Event Examples

```csharp
// OrderPlacedIntegrationEvent.cs
public record OrderPlacedIntegrationEvent(
    Guid OrderId,
    Guid CustomerId,
    decimal TotalAmount,
    string Currency,
    IReadOnlyList<OrderItemContract> Items) : IntegrationEvent;

public record OrderItemContract(
    Guid ProductId,
    int Quantity,
    decimal UnitPrice);

// OrderCancelledIntegrationEvent.cs
public record OrderCancelledIntegrationEvent(
    Guid OrderId,
    Guid CustomerId,
    string Reason) : IntegrationEvent;

// OrderShippedIntegrationEvent.cs
public record OrderShippedIntegrationEvent(
    Guid OrderId,
    string TrackingNumber,
    DateTime EstimatedDelivery) : IntegrationEvent;
```

---

## How Integration Events Are Triggered

Integration events are published as a side effect of domain events. The flow inside this service:

```plaintext
1. PlaceOrderCommandHandler calls order.Place()
2. Order raises OrderPlacedEvent (domain event, in-process)
3. SaveChanges writes aggregate + outbox record (same transaction)
4. Domain event handler runs:
       OnOrderPlacedHandler → calls IIntegrationEventPublisher
         (which writes to Outbox — still same transaction)
5. OutboxProcessor (background) reads outbox and publishes
       OrderPlacedIntegrationEvent → message broker
```

The handler that triggers the integration event publish lives in `Features/Orders/EventHandlers/`:

```csharp
// Features/Orders/EventHandlers/PublishOrderPlacedIntegrationEventHandler.cs
public class PublishOrderPlacedIntegrationEventHandler
    : INotificationHandler<OrderPlacedEvent>
{
    private readonly IIntegrationEventPublisher _publisher;

    public PublishOrderPlacedIntegrationEventHandler(IIntegrationEventPublisher publisher)
        => _publisher = publisher;

    public async Task Handle(OrderPlacedEvent notification, CancellationToken ct)
    {
        var integrationEvent = new OrderPlacedIntegrationEvent(
            OrderId: notification.OrderId.Value,
            CustomerId: notification.CustomerId.Value,
            TotalAmount: notification.TotalAmount.Amount,
            Currency: notification.TotalAmount.Currency,
            Items: notification.Items.Select(i => new OrderItemContract(
                i.ProductId.Value, i.Quantity, i.UnitPrice.Amount)).ToList());

        await _publisher.PublishAsync(integrationEvent, ct);
    }
}
```

---

## Contract Ownership and Consumer Copying

This is the most important rule about integration events in a microservices ecosystem:

**OrderService owns `OrderPlacedIntegrationEvent`. NotificationService and InventoryService need it.**

The wrong approach: create a shared NuGet package with the event class and make all services depend on it. This creates compile-time coupling between services — a change to the event shape requires coordinating releases across all consumers simultaneously.

The correct approach: **consumers copy the event class** into their own codebase. The copy is intentional. Each consumer's copy can evolve independently using tolerant reader patterns — ignoring fields they don't need, providing defaults for new fields.

```plaintext
OrderService/
└─ Application/IntegrationEvents/
   └─ OrderPlacedIntegrationEvent.cs   ← source of truth, owned here

NotificationService/
└─ Application/IntegrationEvents/Consumed/
   └─ OrderPlacedIntegrationEvent.cs   ← copied, notification service's version

InventoryService/
└─ Application/IntegrationEvents/Consumed/
   └─ OrderPlacedIntegrationEvent.cs   ← copied, inventory service's version
```

---

## Important Notes

- Keep integration event classes **flat and serialization-friendly**. Avoid nesting complex domain types — use primitives, strings, decimals, and GUIDs. The event must survive serialization to JSON and back without domain logic.
- Add **no behavior** to integration event classes. They are pure data carriers. No methods, no computed properties, no validation logic.
- Version your events carefully. Once published, consumers depend on the shape. Adding optional fields is safe (consumers ignore them). Removing or renaming fields is a breaking change requiring a versioning strategy (`OrderPlacedIntegrationEventV2`).
- In Java, use plain records or POJOs with Jackson annotations for JSON serialization. Avoid framework-specific types in the event class itself.
