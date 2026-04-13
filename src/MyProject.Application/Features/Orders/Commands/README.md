# Features / Orders / Commands

## Purpose

Contains the **input models for write operations** on the Order aggregate. Each command represents a distinct, named intent to change state. Commands are the "verbs" of the CQRS pattern.

## Naming Convention

`{Verb}{Aggregate}Command` — verb first, aggregate second.

```
PlaceOrderCommand
CancelOrderCommand
ConfirmOrderCommand
UpdateOrderShippingAddressCommand
```

## Rules

- Commands are **immutable** — use `record` types in C# or `final` fields in Java
- Commands carry only **primitive data or IDs** needed by the handler — no domain objects, no entities
- One file per command
- Commands implement `ICommand` (no return value) or `ICommand<TResponse>` (returns a typed result via `Result<T>`)
- Command properties use simple types: `Guid`, `string`, `decimal`, `int` — not domain types like `OrderId` or `Money`

## What NOT to put here

- Handler logic (goes in `Handlers/`)
- Validation rules (go in `Validators/`)
- Business logic of any kind
- Domain objects or value objects as properties

## Example

```csharp
// PlaceOrderCommand.cs
public record PlaceOrderCommand(
    Guid CustomerId,
    IReadOnlyList<OrderItemRequest> Items,
    AddressDto ShippingAddress) : ICommand<Guid>;

public record OrderItemRequest(Guid ProductId, int Quantity);
```

## Example files

```
PlaceOrderCommand.cs
CancelOrderCommand.cs
ConfirmOrderCommand.cs
UpdateOrderShippingAddressCommand.cs
```
