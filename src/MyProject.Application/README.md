# MyProject.Application

## Why This Project Exists

The Application layer orchestrates **use cases**. It knows *what needs to happen* to fulfill a business operation, but it delegates *how things happen technically* to the Infrastructure layer through abstractions.

This layer translates between the outside world (commands and queries arriving from the API) and the Domain (business rules and entities). It is the director: it calls repositories to load aggregates, invokes domain methods that enforce business rules, persists changes, and dispatches domain events.

**What this layer never does:**

- It never contains business rules (those live in Domain)
- It never contains technical implementations (those live in Infrastructure)
- It never knows about HTTP, databases, or external APIs directly

---

## What Belongs Here

- **CQRS handlers:** Command handlers (write operations) and query handlers (read operations)
- **Commands and queries:** The input models that handlers process
- **DTOs:** The output models that handlers return
- **Validators:** Input validation rules using FluentValidation
- **Behaviors:** Cross-cutting pipeline logic (logging, validation, transactions, caching)
- **Abstractions:** Interfaces for external services that Infrastructure will implement
- **Exceptions:** Application-flow exceptions (`NotFoundException`, `ForbiddenException`)
- **Mappings:** AutoMapper or Mapster profiles

---

## What Does NOT Belong Here

- Business rules (invariant enforcement belongs in Domain entities)
- EF Core, Dapper, or any ORM code
- HTTP types (`HttpContext`, `IActionResult`)
- `DbContext` or any persistence implementation
- Direct calls to external APIs or services (define an interface here, implement in Infrastructure)

---

## Dependency Rule

```plaintext
MyProject.Application → MyProject.Domain, MyProject.SharedKernel
```

---

## Recommended Naming Conventions

| Type | Convention | Example |
| --- | --- | --- |
| Command | Verb + noun + `Command` | `PlaceOrderCommand`, `CancelOrderCommand` |
| Query | `Get/Find/List` + noun + `Query` | `GetOrderByIdQuery`, `ListOrdersByCustomerQuery` |
| Handler | Same as command/query + `Handler` | `PlaceOrderCommandHandler`, `GetOrderByIdQueryHandler` |
| DTO | Noun + `Dto` | `OrderDto`, `OrderSummaryDto` |
| Validator | Command/Query name + `Validator` | `PlaceOrderCommandValidator` |
| Mapping profile | Feature + `MappingProfile` | `OrderMappingProfile` |
| Abstraction interface | `I` + service description | `IEmailService`, `IStorageService` |
| Application exception | Descriptive + `Exception` | `NotFoundException`, `ForbiddenException` |

---

## Command vs Query

This layer follows CQRS at the handler level:

| Concept | Command | Query |
| --- | --- | --- |
| Intent | Changes state | Returns data |
| Return type | `Result` or `Result<TId>` | `Result<TDto>` |
| Side effects | Yes (domain events, persistence) | None |
| Transaction | Yes | Usually read-only |
| Example | `PlaceOrderCommand` | `GetOrderByIdQuery` |

Commands and queries are separate classes. A handler handles one command or one query — never both.

---

## Pipeline Behaviors

MediatR behaviors form a pipeline around every handler. The order matters:

```plaintext
Request → LoggingBehavior → ValidationBehavior → TransactionBehavior → Handler → Response
```

| Behavior | Purpose |
| --- | --- |
| `LoggingBehavior` | Logs request entry/exit and execution time |
| `ValidationBehavior` | Runs FluentValidation; short-circuits on failure |
| `TransactionBehavior` | Wraps command handlers in a DB transaction |
| `CachingBehavior` | Returns cached result for cacheable queries |

---

## DependencyInjection.cs

This file registers all Application services and must be the **only file** modified when adding new handlers, validators, or behaviors:

```csharp
public static class DependencyInjection
{
    public static IServiceCollection AddApplicationServices(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddMediatR(cfg =>
            cfg.RegisterServicesFromAssembly(Assembly.GetExecutingAssembly()));

        services.AddValidatorsFromAssembly(Assembly.GetExecutingAssembly());

        services.AddTransient(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
        services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
        services.AddTransient(typeof(IPipelineBehavior<,>), typeof(TransactionBehavior<,>));

        services.AddAutoMapper(Assembly.GetExecutingAssembly());

        return services;
    }
}
```

---

## Folder Overview

| Folder | Contents |
| --- | --- |
| `Abstractions/` | Interfaces for external services (email, storage, payment) |
| `Behaviors/` | MediatR pipeline behaviors |
| `DTOs/` | Shared/common data transfer objects |
| `Features/` | Vertical slices — one folder per feature, contains all related objects |
| `Exceptions/` | Application-flow exceptions |
| `Common/` | Shared base classes used within Application |

---

## Java/Spring Equivalent

In Spring Boot, the Application layer typically maps to:

- `@Service` classes as use case handlers
- Command/Query objects as plain POJOs
- `@Validated` + Bean Validation for input validation
- Spring AOP or custom aspects for cross-cutting behaviors
- `ApplicationEventPublisher` for domain event dispatch

The structure remains the same — group by feature under a `features` or `usecases` package.
