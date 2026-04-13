# SharedKernel / Abstractions

## Why This Folder Exists

This folder contains the **base contracts that define what domain objects are**, not what they do. These are the interfaces and abstract base classes that the Domain layer builds upon, and that the Application layer uses to reason about domain objects generically.

Without this layer of abstraction, each layer would define its own notion of "what an entity is," leading to inconsistency and coupling. By centralizing these contracts here, the entire solution speaks the same structural language.

---

## What Belongs Here

- Marker interfaces for domain modeling primitives
- Abstract base classes for entities and value objects
- Contracts that express identity, auditability, or soft deletion
- The `IDomainEvent` contract used by the Domain event system

---

## What Does NOT Belong Here

- Concrete implementations (those go in Domain)
- Framework annotations or attributes
- Business-specific interfaces (those belong in Domain or Application)
- Interfaces that depend on infrastructure types

---

## Recommended File Names and Types

```plaintext
IEntity.cs
IEntity<TId>.cs
IAggregateRoot.cs
IValueObject.cs
IDomainEvent.cs
IAuditableEntity.cs
ISoftDeletable.cs
IHasDomainEvents.cs
```

---

## Recommended Implementations

### `IEntity / IEntity<TId>`

```csharp
// Non-generic for use cases where Id type is unknown
public interface IEntity
{
    object GetId();
}

// Generic for strongly typed identity
public interface IEntity<TId> : IEntity
{
    TId Id { get; }
}
```

### IAggregateRoot

```csharp
// Marker interface — aggregate roots are the only entry point
// into a cluster of domain objects
public interface IAggregateRoot { }
```

### IHasDomainEvents

```csharp
public interface IHasDomainEvents
{
    IReadOnlyCollection<IDomainEvent> DomainEvents { get; }
    void ClearDomainEvents();
}
```

### IDomainEvent

```csharp
public interface IDomainEvent
{
    Guid EventId { get; }
    DateTime OccurredOn { get; }
}
```

### IAuditableEntity

```csharp
public interface IAuditableEntity
{
    DateTime CreatedAt { get; }
    DateTime? UpdatedAt { get; }
    string CreatedBy { get; }
    string? UpdatedBy { get; }
}
```

---

## Naming Conventions

- All files are interfaces — use `I` prefix
- Names describe **what the object is**, not what it does
- Keep names short and precise: `IEntity`, not `IBaseEntityModel`

---

## Important Notes

- These are intentionally minimal. An `IAggregateRoot` as a pure marker interface is correct — it communicates intent to the reader without forcing a base class.
- Avoid turning these into rich base classes at the SharedKernel level. If you need base class behavior (e.g., `DomainEvents` collection management), define that in `Domain/Common/`, not here.
- In Java, these map directly to interfaces. The `@Entity` annotation from JPA is infrastructure — do not bring it into this folder.
