# Domain / Events

## Why This Folder Exists

Domain events capture **facts that happened within the domain** — things that domain experts care about and that other parts of the system may need to react to. `OrderPlaced`, `CustomerRegistered`, `PaymentProcessed` are not technical events — they are business facts.

By raising events inside aggregates and dispatching them after persistence, this folder enables **decoupled side effects**: sending a confirmation email, updating inventory, or notifying another bounded context does not require the `PlaceOrderHandler` to know about any of those reactions.

This folder also contains the `IDomainEventDispatcher` contract — the interface that defines how events are dispatched. The concrete implementation lives in `Infrastructure/Events/`.

---

## What Belongs Here

- Domain event classes (immutable records of what happened)
- The `IDomainEventDispatcher` interface
- Optionally, a base `DomainEvent` abstract class if shared metadata is needed

---

## What Does NOT Belong Here

- Event handler implementations (those go in `Application/Features/{Feature}/EventHandlers/`)
- The concrete dispatcher implementation (that goes in `Infrastructure/Events/`)
- Integration events (events published to external systems via message brokers — those go in Infrastructure)
- Application events (those are MediatR notifications, not domain events)

---

## Recommended Naming Conventions

| Type | Convention | Example |
| --- | --- | --- |
| Domain event class | Past tense noun phrase + `Event` | `OrderPlacedEvent`, `CustomerRegisteredEvent` |
| Event properties | Immutable, set in constructor | All `init` or `readonly` |
| Dispatcher interface | `IDomainEventDispatcher` | Fixed name |

---

## IDomainEventDispatcher

```csharp
// .NET
public interface IDomainEventDispatcher
{
    Task DispatchAsync(
        IReadOnlyCollection<IDomainEvent> domainEvents,
        CancellationToken cancellationToken = default);
}
```

```java
// Java
public interface DomainEventDispatcher {
    void dispatch(List<DomainEvent> domainEvents);
}
```

---

## Domain Event Examples

```csharp
// Base class (optional but useful for metadata)
public abstract record DomainEvent : IDomainEvent
{
    public Guid EventId { get; } = Guid.NewGuid();
    public DateTime OccurredOn { get; } = DateTime.UtcNow;
}

// Concrete events
public record OrderPlacedEvent(
    OrderId OrderId,
    CustomerId CustomerId,
    Money TotalAmount) : DomainEvent;

public record OrderCancelledEvent(
    OrderId OrderId,
    string Reason) : DomainEvent;

public record OrderItemAddedEvent(
    OrderId OrderId,
    ProductId ProductId,
    int Quantity) : DomainEvent;

public record CustomerRegisteredEvent(
    CustomerId CustomerId,
    EmailAddress Email) : DomainEvent;
```

---

## How Domain Events Flow Through the System

```plaintext
1. Application handler calls aggregate method:
       order.Place();

2. Aggregate raises event internally:
       _domainEvents.Add(new OrderPlacedEvent(Id, CustomerId, TotalAmount));

3. Repository saves aggregate via UoW:
       await _unitOfWork.SaveChangesAsync();

4. IDomainEventDispatcher dispatches after save:
       await _dispatcher.DispatchAsync(order.DomainEvents);

5. Application event handlers react:
       - SendOrderConfirmationEmailHandler
       - DeductInventoryHandler
       - NotifyWarehouseHandler

6. Aggregate clears its events:
       order.ClearDomainEvents();
```

The key insight: step 4 uses `IDomainEventDispatcher`, which is defined here in Domain but implemented in Infrastructure. The aggregate and the handlers never need to know how dispatching works.

---

## Event Granularity

- Raise events that **domain experts would recognize**. "An order was placed" is meaningful. "An order's status field was set to 3" is not.
- Raise events **after invariants are satisfied**, not before. The event announces that something valid happened.
- Prefer **fine-grained events** over coarse ones. `OrderItemAddedEvent` is more useful than `OrderUpdatedEvent`.

---

## Domain Events vs Integration Events

| Concern | Domain Events | Integration Events |
| --- | --- | --- |
| Audience | Same process, same bounded context | External services, other bounded contexts |
| Transport | In-memory dispatcher | Message broker (RabbitMQ, Kafka, Azure Service Bus) |
| Location | `Domain/Events/` | `Infrastructure/Integrations/` |
| Timing | After DB save, before response | After DB commit, async |
| Guaranteed delivery | No (process crash loses them) | Yes (outbox pattern) |

---

## Important Notes

- Domain events are **immutable**. Once raised, their data never changes. Use `record` in C# or `final` fields in Java.
- Do not reference infrastructure types in domain events. An event carries only domain-typed data (ids, value objects, primitives).
- The `IDomainEventDispatcher` contract lives here in Domain because aggregates and repositories need to reference it. The implementation is in Infrastructure to keep the Domain dependency-free from frameworks.
- In Java/Spring, domain event dispatching is commonly done via `ApplicationEventPublisher` in the infrastructure adapter layer, implementing a custom `DomainEventDispatcher` interface defined here.
