# Application / Features

## Why This Folder Exists

This folder organizes all use case logic using a **Vertical Slice** approach. Each business feature gets its own subfolder containing everything related to that feature — commands, queries, handlers, validators, and mappings — in one place.

The alternative (horizontal organization by type: all commands in one folder, all handlers in another) forces developers to navigate across multiple folders to understand a single use case. Vertical slices keep everything cohesive and co-located.

**Guiding question:** If a developer asks "what happens when an order is placed?", they should find the complete answer in `Features/Orders/` — not spread across `Commands/`, `Handlers/`, `Validators/`, and `Mappings/` at the top level.

---

## What Belongs Here

- One subfolder per business feature or aggregate
- Inside each feature folder: Commands, Queries, Handlers, Validators, Mappings, and optionally EventHandlers

---

## What Does NOT Belong Here

- Cross-cutting behaviors (those go in `Behaviors/`)
- Shared DTOs used across multiple features (those go in `DTOs/`)
- Domain logic (that belongs in `Domain/`)
- Infrastructure implementations

---

## Folder Structure Per Feature

```
Features/
├─ Orders/
│  ├─ Commands/
│  │  ├─ PlaceOrderCommand.cs
│  │  └─ CancelOrderCommand.cs
│  ├─ Queries/
│  │  ├─ GetOrderByIdQuery.cs
│  │  └─ ListOrdersByCustomerQuery.cs
│  ├─ Handlers/
│  │  ├─ PlaceOrderCommandHandler.cs
│  │  ├─ CancelOrderCommandHandler.cs
│  │  ├─ GetOrderByIdQueryHandler.cs
│  │  └─ ListOrdersByCustomerQueryHandler.cs
│  ├─ Validators/
│  │  ├─ PlaceOrderCommandValidator.cs
│  │  └─ CancelOrderCommandValidator.cs
│  ├─ Mappings/
│  │  └─ OrderMappingProfile.cs
│  ├─ EventHandlers/          // optional
│  │  └─ SendOrderConfirmationOnOrderPlacedHandler.cs
│  └─ DTOs/                   // optional: feature-specific DTOs
│     ├─ OrderDto.cs
│     └─ OrderSummaryDto.cs
│
├─ Customers/
│  └─ ...
│
└─ Products/
   └─ ...
```

---

## Naming Conventions Per Feature

| Type | Convention | Example |
| --- | --- | --- |
| Feature folder | PascalCase plural noun | `Orders`, `Customers`, `Products` |
| Command | Verb + noun + `Command` | `PlaceOrderCommand` |
| Query | `Get/List/Find` + noun + optional filter + `Query` | `GetOrderByIdQuery`, `ListOrdersByCustomerQuery` |
| Handler | Full command/query name + `Handler` | `PlaceOrderCommandHandler` |
| Validator | Command/Query name + `Validator` | `PlaceOrderCommandValidator` |
| Mapping profile | Feature + `MappingProfile` | `OrderMappingProfile` |
| Event handler | Action + `On` + EventName + `Handler` | `SendConfirmationOnOrderPlacedHandler` |

---

## When to Create a New Feature Folder

Create a new folder when:
- A new aggregate is introduced (`Products/`, `Invoices/`)
- A sufficiently distinct business capability warrants separation (`Reporting/`, `Notifications/`)

Do **not** create a folder for:
- A single command with no related queries
- Infrastructure concerns (batch jobs, scheduled tasks)

For cross-aggregate operations (e.g., "checkout" that touches Orders, Inventory, and Payments), use a dedicated feature folder like `Checkout/` rather than fragmenting the logic across aggregate-named folders.

---

## Important Notes

- Feature folders grow organically. Start with just Commands and Handlers. Add Validators when validation logic becomes complex. Add Mappings when profiles are needed. Don't pre-create empty folders.
- If a feature folder grows beyond ~10 handler files, consider whether it represents multiple distinct sub-domains that should be separated.
- In Java/Spring, the equivalent structure uses packages: `features.orders.commands`, `features.orders.queries`, etc.
