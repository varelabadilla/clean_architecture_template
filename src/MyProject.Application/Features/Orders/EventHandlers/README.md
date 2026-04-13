# Features / Orders / EventHandlers

## Why This Folder Exists

This folder contains two types of event handlers that did not exist in v0.2:

1. **Integration event publishers** — domain event handlers whose job is to translate a domain event into an integration event and hand it to `IIntegrationEventPublisher`. These run in-process as part of the domain event dispatch cycle.

2. **Integration event consumers** — handlers that react to integration events arriving *from other services* via the message broker. For example, when `InventoryService` publishes a `StockDepletedIntegrationEvent`, an `OrderService` consumer here may automatically cancel affected pending orders.

---

## What Belongs Here

- Handlers that publish integration events in response to domain events
- Handlers that consume integration events from other services
- Locally-copied consumed event contracts (the event classes received from other services)

---

## What Does NOT Belong Here

- Domain event handlers with pure in-process side effects (those can stay in a general `EventHandlers/` folder or alongside handlers)
- Integration event class definitions *published by this service* (those live in `Application/IntegrationEvents/`)
- MassTransit consumer configuration (that lives in `Infrastructure/Messaging/Consumers/`)

---

## Naming Conventions

| Type | Convention | Example |
| --- | --- | --- |
| Integration event publisher handler | `Publish` + EventName + `Handler` | `PublishOrderPlacedIntegrationEventHandler` |
| Integration event consumer handler | Action + `On` + EventName + `Handler` | `CancelOrdersOnStockDepletedHandler` |
| Consumed event contract (copied) | Source service name prefix + event name | `InventoryStockDepletedIntegrationEvent` |

---

## Publisher Handler Example

Runs when the domain event `OrderPlacedEvent` is dispatched in-process. Its only job is to translate and publish the integration event:

```csharp
// PublishOrderPlacedIntegrationEventHandler.cs
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
            Items: notification.Items
                .Select(i => new OrderItemContract(
                    i.ProductId.Value, i.Quantity, i.UnitPrice.Amount))
                .ToList());

        await _publisher.PublishAsync(integrationEvent, ct);
    }
}
```

---

## Consumer Handler Example

Reacts to an event published by another service (e.g. `InventoryService`). The consumed event contract is a local copy of the other service's integration event:

```csharp
// Consumed event contract — copied from InventoryService, owned locally
// ConsumedEvents/InventoryStockDepletedIntegrationEvent.cs
public record InventoryStockDepletedIntegrationEvent(
    Guid ProductId,
    int RemainingStock);

// CancelOrdersOnStockDepletedHandler.cs
public class CancelOrdersOnStockDepletedHandler
    : INotificationHandler<InventoryStockDepletedIntegrationEvent>
{
    private readonly IOrderRepository _orders;

    public CancelOrdersOnStockDepletedHandler(IOrderRepository orders)
        => _orders = orders;

    public async Task Handle(
        InventoryStockDepletedIntegrationEvent notification,
        CancellationToken ct)
    {
        var affectedOrders = await _orders.FindAsync(
            new PendingOrdersWithProductSpecification(
                ProductId.From(notification.ProductId)), ct);

        foreach (var order in affectedOrders)
            order.Cancel("Product stock depleted");
    }
}
```

The MassTransit consumer (in `Infrastructure/Messaging/Consumers/`) receives the raw broker message, deserializes it, and dispatches it to MediatR — which routes it to this handler. This handler stays clean of any broker-specific types.

---

## The Bridge Pattern (Infrastructure → Application)

The connection between the broker consumer and this handler follows a deliberate pattern:

```plaintext
Broker message arrives
  → Infrastructure/Messaging/Consumers/StockDepletedConsumer.cs
    → Deserializes to InventoryStockDepletedIntegrationEvent
      → Calls _mediator.Publish(integrationEvent)
        → MediatR routes to CancelOrdersOnStockDepletedHandler (here)
```

This keeps all broker-specific code (MassTransit `IConsumer<T>`, connection handling, dead letter queues) in Infrastructure. The Application handler is a clean MediatR notification handler that can be unit tested without a broker.

---

## Folder Structure

```plaintext
EventHandlers/
├─ PublishOrderPlacedIntegrationEventHandler.cs
├─ PublishOrderCancelledIntegrationEventHandler.cs
├─ PublishOrderShippedIntegrationEventHandler.cs
└─ ConsumedEvents/
   ├─ InventoryStockDepletedIntegrationEvent.cs   ← copied from InventoryService
   ├─ CancelOrdersOnStockDepletedHandler.cs
   └─ PaymentRefundedIntegrationEvent.cs          ← copied from PaymentService
```

---

## Important Notes

- Consumer handlers in this folder are **unit testable** — they depend only on domain repositories and application abstractions. No broker setup needed in tests.
- Consumed event contracts are **local copies**, not shared library references. When `InventoryService` changes its event shape, update the copy here deliberately — do not create an automatic coupling.
- In Java/Spring, publisher handlers are `@EventListener` or `ApplicationListener<T>` beans. Consumer handlers are `@StreamListener` or `@KafkaListener` methods that forward to the application service — the same bridge pattern applies.
