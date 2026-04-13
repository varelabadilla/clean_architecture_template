# Infrastructure / Events

## Why This Folder Exists

This folder contains the **concrete implementation of the domain event dispatcher**. The contract `IDomainEventDispatcher` is defined in `Domain/Events/` — here is where that contract is fulfilled using the actual dispatching mechanism (MediatR, Spring's `ApplicationEventPublisher`, or a custom bus).

The Domain layer defines the contract. Infrastructure provides the engine.

---

## What Belongs Here

- `DomainEventDispatcher.cs` — the concrete implementation of `IDomainEventDispatcher`
- Any infrastructure-level event serialization or routing helpers

---

## What Does NOT Belong Here

- Event handler implementations (those live in `Application/Features/{Feature}/EventHandlers/`)
- Integration event publishers (those belong in `Integrations/`)
- Domain event class definitions (those live in `Domain/Events/`)

---

## Recommended File Names

```plaintext
DomainEventDispatcher.cs
```

---

## Implementation

```csharp
// .NET — using MediatR to dispatch domain events as notifications
public class DomainEventDispatcher : IDomainEventDispatcher
{
    private readonly IPublisher _publisher;
    private readonly ILogger<DomainEventDispatcher> _logger;

    public DomainEventDispatcher(IPublisher publisher, ILogger<DomainEventDispatcher> logger)
    {
        _publisher = publisher;
        _logger = logger;
    }

    public async Task DispatchAsync(
        IReadOnlyCollection<IDomainEvent> domainEvents,
        CancellationToken cancellationToken = default)
    {
        foreach (var domainEvent in domainEvents)
        {
            _logger.LogDebug(
                "Dispatching domain event {EventType} ({EventId})",
                domainEvent.GetType().Name,
                domainEvent.EventId);

            await _publisher.Publish(domainEvent, cancellationToken);
        }
    }
}
```

```java
// Java — using Spring's ApplicationEventPublisher
@Component
public class DomainEventDispatcherImpl implements DomainEventDispatcher {

    private final ApplicationEventPublisher publisher;

    public DomainEventDispatcherImpl(ApplicationEventPublisher publisher) {
        this.publisher = publisher;
    }

    @Override
    public void dispatch(List<DomainEvent> domainEvents) {
        domainEvents.forEach(event -> {
            log.debug("Dispatching domain event: {}", event.getClass().getSimpleName());
            publisher.publishEvent(event);
        });
    }
}
```

---

## Important Notes

- Domain events are dispatched **after** `SaveChangesAsync` completes. This ensures the aggregate is persisted before any side effects run. If a domain event handler fails after persistence, consider using the Outbox pattern for guaranteed delivery.
- MediatR's `IPublisher.Publish` dispatches to all `INotificationHandler<T>` implementations. This means Application event handlers are automatically discovered without any wiring in this class.
- In production systems with strict consistency requirements, replace this in-memory dispatcher with an Outbox pattern implementation: events are written to an `outbox` table within the same transaction, then a background worker publishes them to the message broker.
