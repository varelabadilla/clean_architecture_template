# Infrastructure / Messaging / Consumers

## Purpose

Contains one consumer class per **integration event type consumed from other services**. Each consumer receives a message from the broker, performs minimal validation, and forwards the event to MediatR (or Spring's `ApplicationEventPublisher`) for the Application layer to handle.

## Naming Convention

`{EventName}Consumer` — e.g., `StockDepletedConsumer`, `PaymentRefundedConsumer`

## Rules

- One consumer per event type
- Consumers contain zero business logic — they only deserialize and forward
- All retry, dead letter, and error handling configuration is done here or in `DependencyInjection.cs`
- Consumers are registered automatically via `AddConsumers(Assembly.GetExecutingAssembly())` in MassTransit

## What to put in each consumer

1. Log receipt of the message (structured log with relevant IDs)
2. Forward to MediatR via `_mediator.Publish(message)`
3. Nothing else

## Example files

```plaintext
StockDepletedConsumer.cs         ← from InventoryService
PaymentRefundedConsumer.cs       ← from PaymentService
CustomerDeletedConsumer.cs       ← from CustomerService
```
