# Application / Features / Orders

## What This Feature Covers

This folder contains all use cases related to the **Order** aggregate. Every command, query, handler, validator, and mapping that involves reading or modifying order data lives here.

This is a **reference implementation** folder — it serves as the pattern that all other feature folders should follow.

---

## Subfolder Responsibilities

### Commands/

Input models for write operations. A command expresses **intent to change state**.

```csharp
// PlaceOrderCommand.cs
public record PlaceOrderCommand(
    Guid CustomerId,
    IReadOnlyList<OrderItemRequest> Items,
    AddressDto ShippingAddress) : ICommand<Guid>;

// CancelOrderCommand.cs
public record CancelOrderCommand(
    Guid OrderId,
    string CancellationReason) : ICommand;

// Supporting nested type for PlaceOrderCommand
public record OrderItemRequest(
    Guid ProductId,
    int Quantity);
```

### Queries/

Input models for read operations. A query expresses **intent to read data**.

```csharp
// GetOrderByIdQuery.cs
public record GetOrderByIdQuery(Guid OrderId) : IQuery<OrderDetailDto>;

// ListOrdersByCustomerQuery.cs
public record ListOrdersByCustomerQuery(
    Guid CustomerId,
    int Page = 1,
    int PageSize = 20) : IQuery<PagedResult<OrderSummaryDto>>;
```

### Handlers/

The classes that execute commands and queries. One handler per command or query.

```csharp
// PlaceOrderCommandHandler.cs
public class PlaceOrderCommandHandler
    : IRequestHandler<PlaceOrderCommand, Result<Guid>>
{
    private readonly IOrderRepository _orders;
    private readonly IProductRepository _products;

    public PlaceOrderCommandHandler(
        IOrderRepository orders,
        IProductRepository products)
    {
        _orders = orders;
        _products = products;
    }

    public async Task<Result<Guid>> Handle(
        PlaceOrderCommand command,
        CancellationToken cancellationToken)
    {
        var customerId = CustomerId.From(command.CustomerId);
        var order = new Order(OrderId.New(), customerId);

        foreach (var itemRequest in command.Items)
        {
            var product = await _products.GetByIdAsync(
                ProductId.From(itemRequest.ProductId), cancellationToken);

            if (product is null)
                return Result<Guid>.Failure(
                    Error.NotFound("Product.NotFound", $"Product '{itemRequest.ProductId}' not found."));

            order.AddItem(product.Id, itemRequest.Quantity, product.Price);
        }

        order.Place();
        await _orders.AddAsync(order, cancellationToken);

        return Result<Guid>.Success(order.Id.Value);
    }
}
```

### Validators/

FluentValidation validators for commands. Queries rarely need validators unless they have complex filter constraints.

```csharp
// PlaceOrderCommandValidator.cs
public class PlaceOrderCommandValidator : AbstractValidator<PlaceOrderCommand>
{
    public PlaceOrderCommandValidator()
    {
        RuleFor(x => x.CustomerId)
            .NotEmpty().WithMessage("Customer ID is required.");

        RuleFor(x => x.Items)
            .NotEmpty().WithMessage("Order must contain at least one item.");

        RuleForEach(x => x.Items).ChildRules(item =>
        {
            item.RuleFor(i => i.ProductId)
                .NotEmpty().WithMessage("Product ID is required.");

            item.RuleFor(i => i.Quantity)
                .GreaterThan(0).WithMessage("Quantity must be greater than zero.");
        });

        RuleFor(x => x.ShippingAddress).NotNull()
            .WithMessage("Shipping address is required.");
    }
}
```

### Mappings/

AutoMapper or Mapster profiles for mapping domain objects to DTOs.

```csharp
// OrderMappingProfile.cs
public class OrderMappingProfile : Profile
{
    public OrderMappingProfile()
    {
        CreateMap<Order, OrderSummaryDto>()
            .ForMember(d => d.TotalAmount, o => o.MapFrom(s => s.TotalAmount.Amount))
            .ForMember(d => d.Currency, o => o.MapFrom(s => s.TotalAmount.Currency))
            .ForMember(d => d.Status, o => o.MapFrom(s => s.Status.ToString()));

        CreateMap<Order, OrderDetailDto>()
            .ForMember(d => d.Items, o => o.MapFrom(s => s.Items));

        CreateMap<OrderItem, OrderItemDto>()
            .ForMember(d => d.UnitPrice, o => o.MapFrom(s => s.UnitPrice.Amount));
    }
}
```

### EventHandlers/ (optional)

Handlers that react to domain events raised during command execution.

```csharp
// SendOrderConfirmationOnOrderPlacedHandler.cs
public class SendOrderConfirmationOnOrderPlacedHandler
    : INotificationHandler<OrderPlacedEvent>
{
    private readonly IEmailService _email;

    public SendOrderConfirmationOnOrderPlacedHandler(IEmailService email)
        => _email = email;

    public async Task Handle(
        OrderPlacedEvent notification,
        CancellationToken cancellationToken)
    {
        await _email.SendTemplatedAsync(
            to: notification.CustomerEmail,
            templateId: "order-confirmation",
            templateData: new { OrderId = notification.OrderId },
            cancellationToken: cancellationToken);
    }
}
```

---

## File Naming Summary

```plaintext
Commands/
  PlaceOrderCommand.cs
  CancelOrderCommand.cs
  ConfirmOrderCommand.cs

Queries/
  GetOrderByIdQuery.cs
  ListOrdersByCustomerQuery.cs
  GetOrderStatusQuery.cs

Handlers/
  PlaceOrderCommandHandler.cs
  CancelOrderCommandHandler.cs
  ConfirmOrderCommandHandler.cs
  GetOrderByIdQueryHandler.cs
  ListOrdersByCustomerQueryHandler.cs

Validators/
  PlaceOrderCommandValidator.cs
  CancelOrderCommandValidator.cs

Mappings/
  OrderMappingProfile.cs

EventHandlers/
  SendOrderConfirmationOnOrderPlacedHandler.cs
  DeductInventoryOnOrderPlacedHandler.cs
```

---

## Important Notes

- Handlers should be **thin orchestrators**: load aggregate → call domain method → persist. Business logic belongs in the domain entity.
- A handler that contains `if/else` business logic is a signal that logic should be moved into the aggregate or a domain service.
- Commands return `Result` or `Result<TId>`. Queries return `Result<TDto>` or `Result<PagedResult<TDto>>`.
- Do not share handler classes between commands and queries. One handler per request type, always.
