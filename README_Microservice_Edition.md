# Clean Architecture v0.3 — Microservice Edition

## What This Version Is

v0.3 is a **direct extension of v0.2** designed for the reality of microservices. Every principle, folder, and decision from v0.2 applies here unchanged. v0.3 adds only the pieces that become necessary when a service no longer lives alone — when it must publish facts to the outside world, react to facts from other services, and communicate with other services over a network.

The mental model remains the same: **each microservice is a micro-monolith**. It has its own repository, its own database, its own deployment pipeline, and its own complete v0.2 internal structure. What changes is the Infrastructure layer, which gains the ability to speak the language of distributed systems.

---

## What Changes From v0.2

| Concern | v0.2 (monolith) | v0.3 (microservice) |
| --- | --- | --- |
| Cross-module communication | In-memory MediatR calls | Async messages via broker |
| Domain events | Dispatched in-process | Also published as Integration Events to broker |
| Side effects | Always consistent (same DB transaction) | Eventually consistent (async consumers) |
| Inter-service calls | Not applicable | HTTP clients with retry + circuit breaker |
| Event delivery guarantee | Not needed | Outbox pattern (write-ahead event log) |
| Shared code | SharedKernel in same solution | Nothing shared — each service is autonomous |

---

## Project Map

```plaintext
MyProject/                               (rename to your service name, e.g. OrderService)
├─ microservices-v0.3/
│  ├─ src/
│  │  ├─ MyProject.SharedKernel/         ← unchanged from v0.2 (internal only)
│  │  ├─ MyProject.Domain/               ← unchanged from v0.2
│  │  ├─ MyProject.Application/          ← v0.2 + IntegrationEvents/ + EventHandlers/
│  │  ├─ MyProject.Infrastructure/       ← v0.2 + Messaging/ + Outbox/
│  │  └─ MyProject.Api/                  ← unchanged from v0.2
│  │
│  └─ tests/
│     ├─ MyProject.Domain.Tests/         ← unchanged from v0.2
│     ├─ MyProject.Application.Tests/    ← unchanged from v0.2
│     ├─ MyProject.Infrastructure.Tests/ ← adds messaging and outbox tests
│     ├─ MyProject.Api.Tests/            ← unchanged from v0.2
│     └─ MyProject.Architecture.Tests/   ← adds messaging boundary rules
│
├─ README.md                             ← this file
└─ docker-compose.yml                    ← local broker + DB for development
```

---

## The Two New Concepts in v0.3

### 1. Integration Events

A **domain event** is a fact that happened inside this service, dispatched in-process. An **integration event** is that same fact published to the outside world via a message broker so other services can react to it.

They are different things and must be kept separate:

```plaintext
Domain event:       OrderPlacedEvent       → in-process, MediatR, same transaction
Integration event:  OrderPlacedIntegrationEvent → message broker, async, other services
```

The flow is:

```plaintext
Aggregate raises OrderPlacedEvent
  → Domain event handler runs in-process (e.g. updates read model)
  → Outbox record is written to DB (same transaction as the aggregate save)
    → Outbox processor picks it up and publishes OrderPlacedIntegrationEvent to broker
      → Other services (NotificationService, InventoryService) consume it
```

This two-step publish (write to Outbox → publish from Outbox) is what guarantees that events are never lost even if the process crashes after the DB commit.

### 2. Outbox Pattern

The Outbox pattern solves the **dual write problem**: you cannot atomically write to a database AND publish to a message broker in a single transaction. If you write to the DB and then the process crashes before publishing, the event is lost. If you publish first and the DB write fails, you have published a lie.

The solution: write the event to an `outbox` table **inside the same database transaction** as the aggregate. A background processor reads unpublished outbox records and publishes them to the broker, then marks them as published. The worst case is duplicate delivery (at-least-once), not lost delivery.

```plaintext
┌─────────────────────────────────────┐
│         DB Transaction              │
│  ┌──────────┐    ┌───────────────┐  │
│  │ orders   │    │ outbox_events │  │
│  │ (saved)  │    │ (event saved) │  │
│  └──────────┘    └───────────────┘  │
└─────────────────────────────────────┘
           ↓ (after commit)
  ┌─────────────────────┐
  │  OutboxProcessor    │  ← background worker
  │  reads + publishes  │
  └─────────────────────┘
           ↓
  ┌─────────────────────┐
  │   Message Broker    │  ← RabbitMQ / Kafka
  └─────────────────────┘
           ↓
  Other services consume
```

---

## Dependency Rule (unchanged + extended)

```plaintext
SharedKernel  ← (no dependencies)
Domain        ← SharedKernel
Application   ← Domain, SharedKernel
Infrastructure← Application, Domain, SharedKernel
Api           ← Application, Infrastructure (DI only), SharedKernel
```

The rule does not change. What changes is that `Infrastructure/Messaging/` is a new subfolder inside Infrastructure — it still depends inward, never outward.

---

## Inter-Service Communication Patterns

v0.3 supports two communication styles. Choose based on the use case:

### Asynchronous (preferred)

Used when the caller does not need an immediate response and consistency can be eventual.

```plaintext
OrderService publishes → OrderPlacedIntegrationEvent → broker
NotificationService consumes → sends confirmation email
InventoryService consumes → deducts stock
```

### Synchronous HTTP (when necessary)

Used when the caller needs an immediate answer and the dependency is acceptable (e.g. a price check before placing an order).

Always wrap with resilience policies — never make a raw `HttpClient` call to another service:

```csharp
// .NET — Polly via Microsoft.Extensions.Http.Resilience
builder.Services.AddHttpClient<IInventoryServiceClient, InventoryServiceClient>()
    .AddStandardResilienceHandler(); // retry + circuit breaker + timeout
```

```java
// Java — Resilience4j
@FeignClient(name = "inventory-service", configuration = ResilienceConfig.class)
public interface InventoryServiceClient {
    @GetMapping("/api/stock/{productId}")
    StockResponse getStock(@PathVariable UUID productId);
}
```

---

## Technology Stack

### .NET

| Concern | Library |
|---|---|
| Message bus | MassTransit |
| Broker | RabbitMQ (dev) / Azure Service Bus (prod) |
| Outbox | MassTransit Outbox (built-in) or custom implementation |
| HTTP resilience | Microsoft.Extensions.Http.Resilience (Polly) |
| Background jobs | `IHostedService` / Hangfire |

### Java / Spring

| Concern | Library |
|---|---|
| Message bus | Spring Cloud Stream |
| Broker | RabbitMQ (dev) / Apache Kafka (prod) |
| Outbox | Debezium CDC or custom `@Transactional` outbox table |
| HTTP resilience | Resilience4j + OpenFeign |
| Background jobs | Spring Scheduler / Quartz |

> These are the recommended defaults. Any broker supported by MassTransit (Kafka, SQS, Azure Service Bus) or Spring Cloud Stream (RabbitMQ, Kafka, Kinesis) works with this structure without changing the application layer.

---

## What Each Service Owns Independently

This is the autonomy contract every microservice in the ecosystem must respect:

| Resource | Owned by | Shared? |
| --- | --- | --- |
| Database / schema | This service only | Never |
| Domain model | This service only | Never |
| Integration event contracts | Published by this service | Consumers copy the contract — no shared library |
| API contracts | This service only | Documented via OpenAPI |
| Message broker topics/queues | Named by publishing service | Consumers subscribe — no ownership transfer |

### On integration event contracts

The most common mistake in microservices is creating a shared NuGet/Maven package with integration event classes and making all services depend on it. This reintroduces tight coupling — one change to the package requires coordinating deployments across all consumers.

The correct approach: the publishing service owns the event shape. Consumers **copy** the relevant event class into their own codebase. This is intentional duplication, not a mistake. It keeps each service deployable independently.

---

## Local Development Setup

A `docker-compose.yml` at the root of the service provides the local development environment:

```yaml
# docker-compose.yml
services:
  db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      SA_PASSWORD: "Dev_Password123"
      ACCEPT_EULA: "Y"
    ports:
      - "1433:1433"

  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"    # AMQP
      - "15672:15672"  # Management UI (guest/guest)

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
```

---

## Version History

| Version | Description |
| --- | --- |
| v0.1 | Initial structure: Domain, Application, Infrastructure, Api |
| v0.2 | SharedKernel, explicit IoC, Domain exceptions, IDomainEventDispatcher, ISpecification&lt;T&gt;, Behaviors subdivision, Architecture.Tests |
| v0.3 | Microservice-ready: Integration Events, Messaging layer, Outbox pattern, inter-service HTTP clients with resilience, updated Architecture tests |
