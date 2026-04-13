# Features / Orders / Handlers

## Purpose

Contains the **handler classes** that execute commands and queries. Each handler is the entry point for one use case: it loads the necessary data, invokes domain methods, persists changes (commands only), and returns a result.

## Naming Convention

Full command or query name + `Handler`:

```
PlaceOrderCommandHandler
CancelOrderCommandHandler
GetOrderByIdQueryHandler
ListOrdersByCustomerQueryHandler
```

## Rules

- **One handler per command or query** — never share a handler class between two request types
- Handlers are **thin orchestrators**: load → call domain → persist → return
- **Business logic does NOT live in handlers** — if a handler contains branching business rules (`if order is eligible for X then...`), that logic belongs in a domain entity or domain service
- Command handlers return `Result` or `Result<T>` using the result pattern — never throw for expected failures
- Query handlers return `Result<TDto>` — always map to a DTO before returning, never return domain entities

## The Handler's Responsibility Boundary

A handler answers the question: *"How do I orchestrate the domain to fulfill this use case?"*

It does NOT answer: *"What are the business rules?"* (that's the domain's job) or *"How do I persist to SQL?"* (that's the repository's job).

```csharp
// CORRECT: thin orchestration
public async Task<Result<Guid>> Handle(PlaceOrderCommand command, CancellationToken ct)
{
    var product = await _products.GetByIdAsync(ProductId.From(command.ProductId), ct);
    if (product is null) return Result<Guid>.Failure(Error.NotFound("Product.NotFound", "..."));

    var order = new Order(OrderId.New(), CustomerId.From(command.CustomerId));
    order.AddItem(product.Id, command.Quantity, product.Price); // domain enforces rules
    order.Place();                                               // domain enforces rules

    await _orders.AddAsync(order, ct);
    return Result<Guid>.Success(order.Id.Value);
}

// WRONG: business logic leaking into the handler
public async Task<Result<Guid>> Handle(PlaceOrderCommand command, CancellationToken ct)
{
    if (command.Quantity <= 0)                           // belongs in validator
        return Result<Guid>.Failure(...);
    if (command.CustomerId == Guid.Empty)                // belongs in validator
        return Result<Guid>.Failure(...);
    // checking stock here instead of in domain entity — wrong layer
    var stock = await _inventory.GetStockAsync(command.ProductId, ct);
    if (stock < command.Quantity)
        return Result<Guid>.Failure(...);
    ...
}
```

## Example files

```
PlaceOrderCommandHandler.cs
CancelOrderCommandHandler.cs
ConfirmOrderCommandHandler.cs
GetOrderByIdQueryHandler.cs
ListOrdersByCustomerQueryHandler.cs
GetOrderStatusQueryHandler.cs
```
