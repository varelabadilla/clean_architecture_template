# Features / Orders / Mappings

## Purpose

Contains **AutoMapper or Mapster mapping profiles** that define how Order domain objects are projected into DTOs returned by query handlers.

## Naming Convention

Feature name + `MappingProfile`:

```plaintext
OrderMappingProfile
```

## Rules

- Mappings are **one-directional: domain → DTO** — never define DTO → domain mappings here
- Mapping from HTTP request contracts to application commands happens in controllers, not here
- Profiles are auto-discovered via `AddAutoMapper(assembly)` — no manual registration needed
- One profile file per aggregate is usually sufficient; split only when profiles grow very large

## Manual vs AutoMapper

Both approaches are valid — choose based on complexity:

**Use AutoMapper** when projecting complex nested aggregates with many child collections and value objects. The profile handles recursive mapping cleanly.

**Use manual mapping** in the handler for simple or performance-sensitive cases. It is more explicit and easier to debug:

```csharp
// Manual — explicit, no magic
var dto = new OrderSummaryDto(
    Id: order.Id.Value,
    Status: order.Status.ToString(),
    TotalAmount: order.TotalAmount.Amount,
    Currency: order.TotalAmount.Currency,
    CreatedAt: order.CreatedAt);
```

Be consistent within a feature: pick one approach and stick with it.

## Example

```csharp
public class OrderMappingProfile : Profile
{
    public OrderMappingProfile()
    {
        CreateMap<Order, OrderSummaryDto>()
            .ForMember(d => d.TotalAmount, o => o.MapFrom(s => s.TotalAmount.Amount))
            .ForMember(d => d.Currency,    o => o.MapFrom(s => s.TotalAmount.Currency))
            .ForMember(d => d.Status,      o => o.MapFrom(s => s.Status.ToString()));

        CreateMap<Order, OrderDetailDto>()
            .ForMember(d => d.Items, o => o.MapFrom(s => s.Items));

        CreateMap<OrderItem, OrderItemDto>()
            .ForMember(d => d.UnitPrice, o => o.MapFrom(s => s.UnitPrice.Amount))
            .ForMember(d => d.Currency,  o => o.MapFrom(s => s.UnitPrice.Currency));
    }
}
```

## Example files

```plaintext
OrderMappingProfile.cs
```
