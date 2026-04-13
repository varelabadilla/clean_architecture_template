# Infrastructure / Messaging / Publishers

## Purpose

Contains the concrete implementation of `IIntegrationEventPublisher` — the abstraction defined in `Application/Abstractions/`. This is the only place in the entire solution that references MassTransit's `IPublishEndpoint` or Spring Cloud Stream's `StreamBridge`.

## Naming Convention

Technology name + `IntegrationEventPublisher` — e.g., `MassTransitIntegrationEventPublisher`

## Rules

- One publisher implementation per messaging technology (you will almost never have more than one)
- The implementation is a thin pass-through to the broker client — no logic, no transformation
- Registered as `IIntegrationEventPublisher` in `DependencyInjection.cs`

## What NOT to put here

- Serialization logic (the broker client handles that)
- Retry logic (that belongs on the broker configuration or the Outbox processor)
- Integration event class definitions (those live in `Application/IntegrationEvents/`)

## Example files

```plaintext
MassTransitIntegrationEventPublisher.cs    ← .NET implementation
```
