# Api / Contracts

## Why This Folder Exists

Contracts are the **public HTTP interface** of the API — the request and response shapes that clients (web apps, mobile apps, third parties) depend on. They are separate from Application DTOs because the HTTP contract and the internal data model serve different purposes and must be allowed to evolve independently.

A `PlaceOrderRequest` is what the HTTP client sends. A `PlaceOrderCommand` is what the Application layer understands. They may look similar today but diverge as the API evolves (versioning, field renaming, backward compatibility).

---

## What Belongs Here

- Request models: what clients send in the request body
- Response models: what the API returns to clients
- Optionally, organized by feature/resource in subfolders

---

## What Does NOT Belong Here

- Application DTOs (those live in `Application/DTOs/` or `Application/Features/`)
- Domain entities or value objects
- Business validation (that belongs in Application validators)
- Models that reference infrastructure types

---

## Recommended Structure

```plaintext
Contracts/
├─ Orders/
│  ├─ PlaceOrderRequest.cs
│  ├─ CancelOrderRequest.cs
│  ├─ OrderResponse.cs
│  └─ OrderSummaryResponse.cs
│
├─ Customers/
│  ├─ RegisterCustomerRequest.cs
│  └─ CustomerResponse.cs
│
└─ Common/
   ├─ PagedResponse.cs
   └─ ErrorResponse.cs
```

---

## Naming Conventions

| Type | Convention | Example |
| --- | --- | --- |
| Request model | Action + Resource + `Request` | `PlaceOrderRequest`, `UpdateCustomerRequest` |
| Response model | Resource + detail level + `Response` | `OrderResponse`, `OrderSummaryResponse` |
| Paged response | `PagedResponse<T>` | `PagedResponse<OrderSummaryResponse>` |
| Error response | `ErrorResponse` | Fixed name |

---

## Examples

```csharp
// PlaceOrderRequest.cs
public record PlaceOrderRequest(
    Guid CustomerId,
    IReadOnlyList<OrderItemRequest> Items,
    AddressRequest ShippingAddress);

public record OrderItemRequest(
    Guid ProductId,
    int Quantity);

public record AddressRequest(
    string Street,
    string City,
    string Country,
    string PostalCode);

// OrderResponse.cs
public record OrderResponse(
    Guid Id,
    Guid CustomerId,
    string Status,
    decimal TotalAmount,
    string Currency,
    IReadOnlyList<OrderItemResponse> Items,
    DateTime CreatedAt);

public record OrderItemResponse(
    Guid ProductId,
    string ProductName,
    int Quantity,
    decimal UnitPrice);

// Common error response for all error cases
public record ErrorResponse(
    string Error,
    string Message,
    IDictionary<string, string[]>? Errors,
    string TraceId);
```

---

## Mapping: Contracts → Commands/Queries

Map from contracts to application types at the controller level:

```csharp
// In OrdersController.cs
var command = new PlaceOrderCommand(
    CustomerId: request.CustomerId,
    Items: request.Items
        .Select(i => new OrderItemRequest(i.ProductId, i.Quantity))
        .ToList(),
    ShippingAddress: new AddressDto(
        request.ShippingAddress.Street,
        request.ShippingAddress.City,
        request.ShippingAddress.Country,
        request.ShippingAddress.PostalCode));
```

Or use an extension method on the contract for readability:

```csharp
// PlaceOrderRequest extension
public static PlaceOrderCommand ToCommand(this PlaceOrderRequest request) =>
    new(request.CustomerId, request.Items.Select(i => i.ToDto()).ToList(), request.ShippingAddress.ToDto());
```

---

## API Versioning Consideration

If the API is versioned, organize contracts by version:

```plaintext
Contracts/
├─ V1/
│  └─ Orders/
│     ├─ PlaceOrderRequest.cs
│     └─ OrderResponse.cs
└─ V2/
   └─ Orders/
      └─ PlaceOrderRequest.cs   // changed fields for v2
```

---

## Important Notes

- Contracts are **public API** — once published, changing them is a breaking change for clients. Be conservative with what you expose. You can always add fields; removing them breaks clients.
- Use `record` types in C# for immutability and clean equality semantics.
- Do not add `[Required]` or validation annotations to contracts — validation belongs in Application validators. Contracts just describe shape.
- In Java/Spring, these are plain Java records or POJOs annotated with `@JsonProperty` for serialization customization. Keep them in a `dto` or `contracts` package under the `web` layer.
