# Domain / Enums

## Why This Folder Exists

Domain enumerations represent **fixed sets of meaningful states or categories** that are part of the business language. `OrderStatus`, `PaymentMethod`, and `UserRole` are concepts your domain experts talk about — they belong in the Domain layer, not scattered across the codebase or defined in infrastructure.

Centralizing them here ensures that status transitions and state checks in entity methods, specifications, and handlers all refer to the same canonical type.

---

## What Belongs Here

- Enumerations that model domain states (`OrderStatus`, `ShipmentStatus`)
- Enumerations that model domain categories (`PaymentMethod`, `ProductType`)
- Enumerations that appear in entity properties or method signatures
- Optionally, strongly-typed enumeration classes (Enumeration pattern) as alternatives to language enums

---

## What Does NOT Belong Here

- Infrastructure-specific codes or flags (HTTP status codes, database column codes)
- UI display values or localization keys
- Enumerations used only for configuration or infrastructure purposes

---

## Recommended Naming Conventions

| Type | Convention | Example |
| --- | --- | --- |
| Enum name | PascalCase, singular noun | `OrderStatus`, `PaymentMethod` |
| Enum values | PascalCase | `Pending`, `Placed`, `Shipped` |
| Enumeration class | Same as enum + optional `Type` suffix | `OrderStatus`, `PaymentMethodType` |

---

## Standard Enum (C#)

```csharp
public enum OrderStatus
{
    Pending = 1,
    Placed = 2,
    Confirmed = 3,
    Shipped = 4,
    Delivered = 5,
    Cancelled = 6,
    Refunded = 7
}
```

> Always assign explicit integer values. Default zero-based indexing leads to confusion when values are stored in a database and the enum is later reordered.

---

## Enumeration Class Pattern (for richer behavior)

When an enum needs associated behavior or metadata, consider the Enumeration pattern instead of a language enum:

```csharp
// .NET
public abstract class Enumeration<T> where T : Enumeration<T>
{
    public int Value { get; }
    public string Name { get; }

    protected Enumeration(int value, string name)
    {
        Value = value;
        Name = name;
    }

    public static T FromValue(int value) =>
        All().FirstOrDefault(e => e.Value == value)
        ?? throw new InvalidOperationException($"No {typeof(T).Name} with value {value}");

    public static IEnumerable<T> All() =>
        typeof(T).GetFields(BindingFlags.Public | BindingFlags.Static)
            .Where(f => f.FieldType == typeof(T))
            .Select(f => (T)f.GetValue(null)!);
}

public class PaymentMethod : Enumeration<PaymentMethod>
{
    public static readonly PaymentMethod CreditCard = new(1, "Credit Card");
    public static readonly PaymentMethod BankTransfer = new(2, "Bank Transfer");
    public static readonly PaymentMethod Crypto = new(3, "Cryptocurrency");

    public bool RequiresManualVerification =>
        this == BankTransfer || this == Crypto;

    private PaymentMethod(int value, string name) : base(value, name) { }
}
```

```java
// Java — use enum with fields and methods
public enum PaymentMethod {
    CREDIT_CARD(1, "Credit Card", false),
    BANK_TRANSFER(2, "Bank Transfer", true),
    CRYPTO(3, "Cryptocurrency", true);

    private final int code;
    private final String displayName;
    private final boolean requiresManualVerification;

    PaymentMethod(int code, String displayName, boolean requiresManualVerification) {
        this.code = code;
        this.displayName = displayName;
        this.requiresManualVerification = requiresManualVerification;
    }

    public boolean requiresManualVerification() {
        return requiresManualVerification;
    }
}
```

---

## State Transition Guidance

If an enum represents **state**, document the valid transitions either in a comment or in a companion class:

```csharp
public static class OrderStatusTransitions
{
    private static readonly Dictionary<OrderStatus, OrderStatus[]> _allowed = new()
    {
        [OrderStatus.Pending]   = [OrderStatus.Placed, OrderStatus.Cancelled],
        [OrderStatus.Placed]    = [OrderStatus.Confirmed, OrderStatus.Cancelled],
        [OrderStatus.Confirmed] = [OrderStatus.Shipped, OrderStatus.Cancelled],
        [OrderStatus.Shipped]   = [OrderStatus.Delivered],
        [OrderStatus.Delivered] = [OrderStatus.Refunded],
        [OrderStatus.Cancelled] = [],
        [OrderStatus.Refunded]  = []
    };

    public static bool CanTransitionTo(OrderStatus current, OrderStatus next) =>
        _allowed.TryGetValue(current, out var allowed) && allowed.Contains(next);
}
```

This logic can also live directly inside the `Order` aggregate — keeping transition rules close to the state they protect.

---

## Important Notes

- Use explicit integer values. When enums are persisted to a database, reordering values without explicit codes causes silent data corruption.
- Avoid using enums as bitmask flags for domain states — this creates combinations that may not have business meaning. If you need multi-value selection, model it explicitly.
- The Enumeration class pattern adds complexity. Use it only when enums need behavior, display names, or metadata. For simple state flags, a language enum is correct.
