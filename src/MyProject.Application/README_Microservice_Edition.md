# MyProject.Application (v0.3)

## What Changed From v0.2

The Application layer in v0.3 is identical to v0.2 with **two additions**:

1. `IntegrationEvents/` — the contracts for events this service publishes to the outside world
2. `EventHandlers/` inside each feature folder — handlers that react to integration events consumed from other services

Everything else — handlers, commands, queries, validators, behaviors, abstractions — is exactly as documented in v0.2.

---

## New Folder Overview

```plaintext
MyProject.Application/
├─ Abstractions/          ← unchanged from v0.2
├─ Behaviors/             ← unchanged from v0.2
├─ DTOs/                  ← unchanged from v0.2
├─ Exceptions/            ← unchanged from v0.2
├─ Common/                ← unchanged from v0.2
├─ IntegrationEvents/     ← NEW: event contracts this service publishes
└─ Features/
   └─ Orders/
      ├─ Commands/        ← unchanged
      ├─ Queries/         ← unchanged
      ├─ Handlers/        ← unchanged
      ├─ Validators/      ← unchanged
      ├─ Mappings/        ← unchanged
      └─ EventHandlers/   ← NEW: handles integration events from other services
```

---

## Dependency Rule

```plaintext
MyProject.Application → MyProject.Domain, MyProject.SharedKernel
```

Unchanged from v0.2. The Application layer still has zero knowledge of Infrastructure, messaging frameworks, or brokers.

---

## Integration Events vs Domain Events — Recap

| | Domain Event | Integration Event |
| --- | --- | --- |
| Defined in | `Domain/Events/` | `Application/IntegrationEvents/` |
| Dispatched by | `IDomainEventDispatcher` (in-process) | Message broker (via Infrastructure) |
| Audience | Same service, same process | Other services, other processes |
| Consistency | Strong (same DB transaction) | Eventual (async) |
| Example | `OrderPlacedEvent` | `OrderPlacedIntegrationEvent` |

The Application layer defines the integration event **contracts** — the shape of the message. Infrastructure publishes them. This keeps the contract definition in the correct layer (Application knows what to communicate) while keeping the technical publishing mechanism in Infrastructure.

---

## IIntegrationEventPublisher Abstraction

The Application layer defines the interface for publishing integration events. Infrastructure implements it:

```csharp
// Application/Abstractions/IIntegrationEventPublisher.cs
public interface IIntegrationEventPublisher
{
    Task PublishAsync<T>(T integrationEvent, CancellationToken cancellationToken = default)
        where T : IntegrationEvent;
}
```

```java
// Java equivalent
public interface IntegrationEventPublisher {
    <T extends IntegrationEvent> void publish(T integrationEvent);
}
```

This means domain event handlers that trigger integration event publishing remain in the Application layer and use this abstraction — they never reference MassTransit, RabbitMQ, or Kafka directly.

---

## Important Notes

- Integration event classes are **owned by the publishing service**. Other services that consume these events copy the class definition into their own codebase — they do not reference this project.
- Do not put integration event handlers here — those are consumers of events from *other* services and live in `Features/{Feature}/EventHandlers/`.
- The `DependencyInjection.cs` in this layer does not register messaging infrastructure — that belongs in `Infrastructure/DependencyInjection.cs`.
