# Application / DTOs

## Why This Folder Exists

Data Transfer Objects (DTOs) are the **output shapes** that handlers return to callers. They represent a projection of domain data tailored for consumption — not the domain entity itself, which carries behavior and invariants that have no meaning outside the domain.

Returning domain entities directly from handlers would couple the API contract to the domain model, making it impossible to evolve either independently. DTOs decouple the domain model from what is exposed.

This folder contains **shared or common DTOs** used across multiple features. Feature-specific DTOs live alongside their handlers in `Features/{Feature}/`.

---

## What Belongs Here

- DTOs that are shared across multiple features or handlers
- Pagination response wrappers
- Common envelope types used across the application

---

## What Does NOT Belong Here

- Feature-specific DTOs (those live in `Features/{Feature}/`)
- API request/response contracts (those go in `Api/Contracts/`)
- Domain entities or value objects
- DTOs with business logic

---

## Recommended Naming Conventions

| Type | Convention | Example |
| --- | --- | --- |
| Read DTO | Noun + `Dto` | `OrderDto`, `CustomerDto` |
| Summary/list DTO | Noun + `SummaryDto` | `OrderSummaryDto` |
| Detail DTO | Noun + `DetailDto` | `OrderDetailDto` |
| Paginated result | `PagedResult<T>` | `PagedResult<OrderSummaryDto>` |

---

## DTO Design Guidelines

### Keep DTOs flat when possible

```csharp
// Prefer flat projection for list views
public record OrderSummaryDto(
    Guid Id,
    string CustomerName,
    decimal TotalAmount,
    string Currency,
    string Status,
    DateTime CreatedAt);

// Use nested DTOs only for detail views that genuinely need nesting
public record OrderDetailDto(
    Guid Id,
    CustomerDto Customer,
    IReadOnlyList<OrderItemDto> Items,
    Money TotalAmount,
    string Status,
    DateTime CreatedAt);
```

### Never expose domain internals

```csharp
// WRONG: Returns the entity — exposes domain behavior and internals
public async Task<Order> Handle(GetOrderByIdQuery query, ...)

// RIGHT: Returns a DTO — safe projection of what callers need
public async Task<Result<OrderDetailDto>> Handle(GetOrderByIdQuery query, ...)
```

---

## Common Shared DTOs

```csharp
// Pagination wrapper
public record PagedResult<T>(
    IReadOnlyList<T> Items,
    int Page,
    int PageSize,
    int TotalCount)
{
    public bool HasNextPage => Page * PageSize < TotalCount;
    public bool HasPreviousPage => Page > 1;
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
}

// Audit fields used across multiple DTOs
public record AuditDto(
    DateTime CreatedAt,
    string CreatedBy,
    DateTime? UpdatedAt,
    string? UpdatedBy);
```

---

## Mapping from Domain to DTO

DTOs are populated by mapping from domain entities, either via AutoMapper profiles (defined in `Features/{Feature}/Mappings/`) or by manual projection in query handlers:

```csharp
// Manual projection (direct and explicit)
var dto = new OrderSummaryDto(
    Id: order.Id.Value,
    CustomerName: order.CustomerName,
    TotalAmount: order.TotalAmount.Amount,
    Currency: order.TotalAmount.Currency,
    Status: order.Status.ToString(),
    CreatedAt: order.CreatedAt);

// AutoMapper profile (in Features/Orders/Mappings/)
public class OrderMappingProfile : Profile
{
    public OrderMappingProfile()
    {
        CreateMap<Order, OrderSummaryDto>()
            .ForMember(d => d.TotalAmount, o => o.MapFrom(s => s.TotalAmount.Amount))
            .ForMember(d => d.Currency, o => o.MapFrom(s => s.TotalAmount.Currency));
    }
}
```

---

## Important Notes

- DTOs are **data containers** — no methods, no business logic, no validation. They are shaped for transport, not for behavior.
- Use `record` types in C# for DTOs — they are immutable by default and provide structural equality, which is exactly the right semantics.
- In Java, DTOs are commonly implemented as plain POJOs, Java records (Java 16+), or Lombok `@Value` classes.
- Avoid "god DTOs" that contain every possible field for every possible use case. Define multiple focused DTOs for different views of the same concept (`OrderSummaryDto`, `OrderDetailDto`, `OrderAuditDto`).
