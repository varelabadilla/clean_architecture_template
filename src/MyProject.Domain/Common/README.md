# Domain / Common

## Why This Folder Exists

This folder contains the **base classes that domain objects inherit from** — the concrete implementations of the abstractions defined in `SharedKernel/Abstractions/`. While SharedKernel defines the contracts (`IEntity`, `IAggregateRoot`), this folder provides the actual base classes with built-in behavior: domain event collection, identity management, and value object equality.

These are the building blocks every entity and value object in this Domain project extends.

---

## What Belongs Here

- `Entity<TId>` base class — identity, equality by ID, domain event collection
- `AggregateRoot<TId>` base class — extends `Entity`, manages domain event list
- `ValueObject` base class — structural equality via `GetEqualityComponents()`
- `AuditableEntity<TId>` — extends `Entity`, adds audit fields
- Domain-wide constants if any exist

---

## What Does NOT Belong Here

- Business logic of any kind
- Concrete domain objects (those go in `Entities/`, `ValueObjects/`, etc.)
- Infrastructure base classes or EF-specific base types
- Application-level abstractions

---

## Recommended File Names

```plaintext
Entity.cs
AggregateRoot.cs
ValueObject.cs
AuditableEntity.cs
```

---

## Recommended Implementations

### `Entity<TId>`

```csharp
public abstract class Entity<TId> : IEntity<TId>, IHasDomainEvents
    where TId : notnull
{
    public TId Id { get; protected set; } = default!;

    private readonly List<IDomainEvent> _domainEvents = new();
    public IReadOnlyCollection<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    protected void AddDomainEvent(IDomainEvent domainEvent) =>
        _domainEvents.Add(domainEvent);

    public void ClearDomainEvents() => _domainEvents.Clear();

    public override bool Equals(object? obj)
    {
        if (obj is not Entity<TId> other) return false;
        if (ReferenceEquals(this, other)) return true;
        if (GetType() != other.GetType()) return false;
        return Id.Equals(other.Id);
    }

    public override int GetHashCode() => HashCode.Combine(GetType(), Id);

    public static bool operator ==(Entity<TId>? left, Entity<TId>? right) =>
        left?.Equals(right) ?? right is null;

    public static bool operator !=(Entity<TId>? left, Entity<TId>? right) =>
        !(left == right);

    public object GetId() => Id;
}
```

### `AggregateRoot<TId>`

```csharp
// AggregateRoot extends Entity but signals aggregate boundary intent
public abstract class AggregateRoot<TId> : Entity<TId>, IAggregateRoot
    where TId : notnull
{
    // Aggregate roots may add cross-cutting behavior here in the future
    // (optimistic concurrency tokens, version tracking, etc.)
}
```

### ValueObject (abstract base)

```csharp
public abstract class ValueObject
{
    protected abstract IEnumerable<object?> GetEqualityComponents();

    public override bool Equals(object? obj)
    {
        if (obj is null || GetType() != obj.GetType()) return false;
        return GetEqualityComponents()
            .SequenceEqual(((ValueObject)obj).GetEqualityComponents());
    }

    public override int GetHashCode() =>
        GetEqualityComponents()
            .Aggregate(0, (hash, component) =>
                HashCode.Combine(hash, component?.GetHashCode() ?? 0));

    public static bool operator ==(ValueObject? left, ValueObject? right) =>
        left?.Equals(right) ?? right is null;

    public static bool operator !=(ValueObject? left, ValueObject? right) =>
        !(left == right);
}
```

### `AuditableEntity<TId>`

```csharp
public abstract class AuditableEntity<TId> : Entity<TId>, IAuditableEntity
    where TId : notnull
{
    public DateTime CreatedAt { get; private set; }
    public DateTime? UpdatedAt { get; private set; }
    public string CreatedBy { get; private set; } = string.Empty;
    public string? UpdatedBy { get; private set; }

    // Called by Infrastructure (e.g., DbContext SaveChanges interceptor)
    public void SetCreated(DateTime createdAt, string createdBy)
    {
        CreatedAt = createdAt;
        CreatedBy = createdBy;
    }

    public void SetUpdated(DateTime updatedAt, string updatedBy)
    {
        UpdatedAt = updatedAt;
        UpdatedBy = updatedBy;
    }
}
```

---

## Java Equivalent

```java
public abstract class Entity<TId> {
    private final TId id;
    private final List<DomainEvent> domainEvents = new ArrayList<>();

    protected Entity(TId id) {
        this.id = Objects.requireNonNull(id, "Id cannot be null");
    }

    public TId getId() { return id; }

    public List<DomainEvent> getDomainEvents() {
        return Collections.unmodifiableList(domainEvents);
    }

    protected void addDomainEvent(DomainEvent event) {
        domainEvents.add(event);
    }

    public void clearDomainEvents() { domainEvents.clear(); }

    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Entity<?> other)) return false;
        return id.equals(other.id) && getClass() == other.getClass();
    }

    @Override
    public int hashCode() { return Objects.hash(getClass(), id); }
}
```

---

## Important Notes

- These base classes intentionally have **no framework imports**. Entity Framework configuration for Id generation, table names, and column mappings is handled in `Infrastructure/Persistence/Configurations/`.
- The `AuditableEntity` audit fields are populated by an infrastructure-level mechanism (an EF `SaveChanges` interceptor, a JPA `@PrePersist/@PreUpdate` listener, or a Unit of Work hook). The domain class declares the fields; infrastructure fills them.
- If you are using C# `record` types for value objects, you do not need the `ValueObject` base class — records provide structural equality natively. Use the base class only when you need custom equality logic or when inheriting from `record` is not practical.
