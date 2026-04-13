# Clean Architecture v0.2

A production-ready project structure based on **Clean Architecture** principles, incorporating **CQRS**, **Domain-Driven Design (DDD)** tactical patterns, and an explicit **Inversion of Control (IoC)** composition strategy.

This template is language-agnostic in concept but the naming conventions and project organization are primarily aligned with **.NET** and **Java/Spring** ecosystems.

---

## Table of Contents

- [Architectural Principles](#architectural-principles)
- [Dependency Rule](#dependency-rule)
- [Project Map](#project-map)
- [Layer Responsibilities](#layer-responsibilities)
- [IoC and Composition Root Strategy](#ioc-and-composition-root-strategy)
- [CQRS and Vertical Slice Organization](#cqrs-and-vertical-slice-organization)
- [Domain Events Flow](#domain-events-flow)
- [Exception Handling Strategy](#exception-handling-strategy)
- [Testing Strategy](#testing-strategy)
- [Technology Mapping](#technology-mapping)
- [What This Structure Is Not](#what-this-structure-is-not)

---

## Architectural Principles

This structure is governed by four core principles:

### 1. The Dependency Rule

Source code dependencies must point **inward only**. Inner layers know nothing about outer layers. This is the single most important rule — every structural decision flows from it.

### 2. Separation of Concerns

Each layer has one clearly defined responsibility. Business rules live in the Domain. Use case orchestration lives in Application. Technical implementations live in Infrastructure. HTTP concerns live in the API layer.

### 3. Explicit Over Implicit

Every architectural boundary is made visible through project separation, not just folder separation. The compiler enforces the dependency rule because cross-layer violations become build errors.

### 4. Testability by Design

Each layer can be tested in isolation. The Domain has zero infrastructure dependencies. The Application layer depends only on abstractions. Infrastructure implementations can be swapped without touching business logic.

---

## Dependency Rule

```plaintext
SharedKernel  ← (no dependencies)
Domain        ← SharedKernel
Application   ← Domain, SharedKernel
Infrastructure← Application, Domain, SharedKernel
Api           ← Application, Infrastructure (only for DI registration), SharedKernel
```

**Key constraint:** `Infrastructure` may reference `Application` to implement its abstractions, but `Application` must never reference `Infrastructure`. The `Api` layer references `Infrastructure` exclusively inside the `DependencyInjection/` folder — never in controllers or business logic.

---

## Project Map

```plaintext
MyProject/
├─ src/
│  ├─ MyProject.SharedKernel/       # Cross-cutting base types, no business logic
│  ├─ MyProject.Domain/             # Enterprise business rules, entities, invariants
│  ├─ MyProject.Application/        # Use case orchestration, CQRS handlers
│  ├─ MyProject.Infrastructure/     # Technical implementations (DB, email, caching...)
│  └─ MyProject.Api/                # HTTP entry point, composition root
│
├─ tests/
│  ├─ MyProject.Domain.Tests/
│  ├─ MyProject.Application.Tests/
│  ├─ MyProject.Infrastructure.Tests/
│  ├─ MyProject.Api.Tests/
│  └─ MyProject.Architecture.Tests/ # Automated dependency rule enforcement
│
├─ Directory.Build.props            # Centralized MSBuild properties (.NET)
├─ Directory.Packages.props         # Central Package Management (.NET)
├─ .editorconfig                    # Code style enforcement
└─ MyProject.sln
```

---

## Layer Responsibilities

### SharedKernel

The foundation layer. Contains only primitive building blocks that have **no dependency on any other layer**. Think of it as the vocabulary every other layer uses to speak. It should never contain business logic, infrastructure concerns, or framework-specific code.

**Examples:** `IEntity<TId>`, `IAggregateRoot`, `IValueObject`, `Result<T>`, `Error`, guard clauses.

### Domain

The heart of the application. Contains **enterprise business rules** — rules that would exist even if there were no software. Entities enforce their own invariants. Value objects model domain concepts. Specifications encode complex query criteria. This layer has zero framework dependencies.

**Examples:** `Order` (aggregate), `Money` (value object), `OrderStatus` (enum), `OrderPlacedEvent` (domain event), `IOrderRepository` (contract).

### Application

Orchestrates use cases. Knows **what needs to happen** but not **how it technically happens**. Uses interfaces defined here or in Domain to remain decoupled from infrastructure. Contains no business rules — those belong in the Domain.

**Examples:** `PlaceOrderCommand`, `PlaceOrderHandler`, `GetOrderByIdQuery`, `ValidationBehavior`, `IEmailService` (abstraction).

### Infrastructure

The technical implementation layer. Knows **how things happen** technically. Implements every interface defined in Domain and Application. This is where Entity Framework, Dapper, SendGrid, Redis, and other frameworks live. Can be replaced without touching Domain or Application.

**Examples:** `OrderRepository`, `AppDbContext`, `EmailService`, `DomainEventDispatcher`, `RedisCacheService`.

### Api

The entry point. Handles HTTP concerns only: routing, model binding, authentication filters, response shaping. Its most important role is acting as the **Composition Root** — the single place where the entire dependency graph is wired together.

**Examples:** `OrdersController`, `GlobalExceptionMiddleware`, `OrderRequest` (contract model), `Program.cs`.

---

## IoC and Composition Root Strategy

A deliberate IoC strategy is one of the key improvements in v0.2. The guiding principle: **each layer owns its own registration, and the Api layer acts as the sole Composition Root**.

### Extension method per layer

Each layer exposes a single extension method:

```csharp
// Application/DependencyInjection.cs
public static IServiceCollection AddApplicationServices(
    this IServiceCollection services, IConfiguration config)
{
    services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(Assembly.GetExecutingAssembly()));
    services.AddValidatorsFromAssembly(Assembly.GetExecutingAssembly());
    services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
    services.AddTransient(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
    return services;
}
```

```csharp
// Infrastructure/DependencyInjection.cs
public static IServiceCollection AddInfrastructureServices(
    this IServiceCollection services, IConfiguration config)
{
    services.AddDbContext<AppDbContext>(options =>
        options.UseSqlServer(config.GetConnectionString("Default")));
    services.AddScoped<IOrderRepository, OrderRepository>();
    services.AddScoped<IDomainEventDispatcher, DomainEventDispatcher>();
    return services;
}
```

### Composition Root (Program.cs)

```csharp
// Api/Program.cs — the only place that knows about all layers
builder.Services
    .AddApplicationServices(builder.Configuration)
    .AddInfrastructureServices(builder.Configuration)
    .AddPresentationServices();
```

This pattern makes the dependency graph **explicit, traceable, and testable**. Each layer's registration can be called independently in integration tests.

### Java/Spring equivalent

In Spring Boot, the same pattern is achieved via `@Configuration` classes per layer:

```java
// application layer
@Configuration
public class ApplicationConfig {
    @Bean
    public OrderCommandHandler orderCommandHandler(IOrderRepository repo) {
        return new OrderCommandHandler(repo);
    }
}

// infrastructure layer
@Configuration
public class InfrastructureConfig {
    @Bean
    public IOrderRepository orderRepository(JpaOrderRepository jpa) {
        return new OrderRepositoryAdapter(jpa);
    }
}
```

---

## CQRS and Vertical Slice Organization

The Application layer uses a **vertical slice** organization under `Features/`. Each business feature owns all its related objects — commands, queries, handlers, validators, and mappings — in one place.

```plaintext
Features/
└─ Orders/
   ├─ Commands/
   │  ├─ PlaceOrderCommand.cs
   │  └─ CancelOrderCommand.cs
   ├─ Queries/
   │  └─ GetOrderByIdQuery.cs
   ├─ Handlers/
   │  ├─ PlaceOrderHandler.cs
   │  ├─ CancelOrderHandler.cs
   │  └─ GetOrderByIdHandler.cs
   ├─ Validators/
   │  └─ PlaceOrderCommandValidator.cs
   └─ Mappings/
      └─ OrderMappingProfile.cs
```

**Why this matters:** When you work on the Orders feature, everything you need is in one folder. You never have to navigate across multiple top-level folders to understand a single use case.

---

## Domain Events Flow

Domain events are raised inside aggregates and dispatched after persistence to maintain consistency.

```plaintext
1. Handler calls aggregate method
       Order.Place(items, customerId)

2. Aggregate raises event internally
       _domainEvents.Add(new OrderPlacedEvent(this.Id))

3. Repository saves aggregate (Infrastructure)
       await _context.SaveChangesAsync()

4. DomainEventDispatcher publishes events (Infrastructure)
       await _dispatcher.DispatchAsync(aggregate.DomainEvents)

5. Event handlers execute (Application/Features/.../EventHandlers)
       SendOrderConfirmationEmailHandler
       UpdateInventoryHandler
```

The contract `IDomainEventDispatcher` lives in `Domain/Events/` so the aggregate never depends on infrastructure. The implementation lives in `Infrastructure/Events/`.

---

## Exception Handling Strategy

Exceptions are divided by layer based on where the rule they enforce lives:

| Exception type | Layer | Examples |
| --- | --- | --- |
| Invariant violations | Domain/Exceptions | `InsufficientStockException`, `InvalidOrderStateException` |
| Use case flow | Application/Exceptions | `NotFoundException`, `ForbiddenException` |
| HTTP translation | Api/Middleware | `GlobalExceptionMiddleware` maps both to HTTP status codes |

**Rule of thumb:** If an entity or value object can throw it, it belongs in Domain. If a handler throws it, it belongs in Application. The API layer never throws domain or application exceptions — it catches and translates them.

---

## Testing Strategy

| Project | Type | Focus |
| --- | --- | --- |
| Domain.Tests | Unit | Entity invariants, value object equality, specification logic |
| Application.Tests | Unit | Handler behavior, validation rules, behavior pipeline |
| Infrastructure.Tests | Integration | Repository queries, EF mappings, external service adapters |
| Api.Tests | Integration/E2E | HTTP contracts, middleware, authentication |
| Architecture.Tests | Static analysis | Dependency rule enforcement via NetArchTest or ArchUnit |

### Architecture test example (.NET)

```csharp
[Fact]
public void Domain_Should_Not_DependOn_Infrastructure()
{
    var result = Types.InAssembly(DomainAssembly)
        .Should().NotHaveDependencyOn("MyProject.Infrastructure")
        .GetResult();

    result.IsSuccessful.Should().BeTrue();
}
```

### Architecture test example (Java — ArchUnit)

```java
@Test
void domain_should_not_depend_on_infrastructure() {
    noClasses()
        .that().resideInAPackage("..domain..")
        .should().dependOnClassesThat()
        .resideInAPackage("..infrastructure..")
        .check(importedClasses);
}
```

---

## Technology Mapping

| Concept | .NET implementation | Java/Spring implementation |
| --- | --- | --- |
| CQRS bus | MediatR | Axon Framework / Spring ApplicationEventPublisher |
| Validation | FluentValidation | Bean Validation (Jakarta) / Hibernate Validator |
| ORM | Entity Framework Core | Spring Data JPA / Hibernate |
| Mapping | AutoMapper / Mapster | MapStruct |
| DI Container | Microsoft.Extensions.DI | Spring IoC Container |
| Architecture tests | NetArchTest | ArchUnit |
| Unit testing | xUnit + FluentAssertions | JUnit 5 + AssertJ |
| Mocking | NSubstitute / Moq | Mockito |

---

## What This Structure Is Not

- **Not microservices:** This is a monolith structure. For microservices, each service would replicate this structure independently.
- **Not CQRS with event sourcing:** CQRS here refers to command/query separation at the handler level, not full event sourcing with an event store.
- **Not a strict Onion Architecture:** While the dependency rule is the same, this structure is more pragmatic and avoids over-abstraction in favor of developer productivity.
- **Not framework-agnostic at the Infrastructure level:** Infrastructure is intentionally coupled to frameworks. That is its purpose. The value is that the coupling is isolated there.

---

## Version History

| Version | Description |
| --- | --- |
| v0.1 | Initial structure suggested: Domain, Application, Infrastructure, Api |
| v0.2 | Added SharedKernel, explicit IoC strategy, Domain exceptions, IDomainEventDispatcher contract, ISpecification&lt;T&gt; base, Behaviors subdivision, Architecture.Tests project |
