# SharedKernel / Guards

## Why This Folder Exists

Guard clauses are **precondition checks placed at the entry point of a method** to fail fast when invalid input is detected. They make invalid state impossible to reach rather than handling it deeper in the call stack.

Centralizing guard logic here avoids duplicating null checks, range validations, and format assertions across the codebase. Every layer — Domain, Application, Infrastructure — can use these guards without creating dependencies between layers.

---

## What Belongs Here

- A static `Guard` class with methods for common precondition checks
- Optionally, a fluent `GuardClause` entry point for readable chaining
- Extension methods on `Guard` if you prefer that pattern

---

## What Does NOT Belong Here

- Business rule validation (that belongs in Domain entities or Application validators)
- Input model validation (that belongs in Application/Behaviors with FluentValidation)
- Error message formatting or localization logic

---

## Recommended File Names

```plaintext
Guard.cs
GuardAgainstNullExtensions.cs    // optional: extension-based guard additions
```

---

## Recommended Implementation

```csharp
public static class Guard
{
    public static T AgainstNull<T>(T? value, string parameterName)
    {
        if (value is null)
            throw new ArgumentNullException(parameterName);
        return value;
    }

    public static string AgainstNullOrEmpty(string? value, string parameterName)
    {
        if (string.IsNullOrEmpty(value))
            throw new ArgumentException("Value cannot be null or empty.", parameterName);
        return value;
    }

    public static string AgainstNullOrWhiteSpace(string? value, string parameterName)
    {
        if (string.IsNullOrWhiteSpace(value))
            throw new ArgumentException("Value cannot be null or whitespace.", parameterName);
        return value;
    }

    public static int AgainstNegative(int value, string parameterName)
    {
        if (value < 0)
            throw new ArgumentOutOfRangeException(parameterName, "Value cannot be negative.");
        return value;
    }

    public static decimal AgainstNegativeOrZero(decimal value, string parameterName)
    {
        if (value <= 0)
            throw new ArgumentOutOfRangeException(parameterName, "Value must be greater than zero.");
        return value;
    }

    public static T AgainstInvalidEnum<T>(T value, string parameterName) where T : Enum
    {
        if (!Enum.IsDefined(typeof(T), value))
            throw new ArgumentOutOfRangeException(parameterName, $"Value '{value}' is not a valid {typeof(T).Name}.");
        return value;
    }
}
```

### Usage in a Domain entity

```csharp
public class Order : AggregateRoot
{
    public Order(Guid id, CustomerId customerId, IReadOnlyList<OrderItem> items)
    {
        Id = Guard.AgainstNull(id, nameof(id));
        CustomerId = Guard.AgainstNull(customerId, nameof(customerId));
        Items = Guard.AgainstNull(items, nameof(items));
    }
}
```

---

## Naming Conventions

| Method pattern | Purpose |
| --- | --- |
| `AgainstNull` | Rejects null values |
| `AgainstNullOrEmpty` | Rejects null or empty strings |
| `AgainstNegative` | Rejects values < 0 |
| `AgainstNegativeOrZero` | Rejects values <= 0 |
| `AgainstInvalidEnum` | Rejects out-of-range enum values |
| `AgainstOutOfRange` | Rejects values outside a min/max range |

Follow the `Against{Condition}` naming pattern consistently. This reads naturally: "guard against null," "guard against negative."

---

## Important Notes

- Guard clauses throw `ArgumentException`, `ArgumentNullException`, or `ArgumentOutOfRangeException` — standard language exceptions for programming contract violations.
- Do not confuse guards with domain validation. A guard enforces **programming contracts** (a parameter must not be null). Domain validation enforces **business rules** (an order must have at least one item). Both matter but they live in different places.
- In Java, this pattern is commonly implemented with `Objects.requireNonNull`, Apache Commons `Validate`, or Guava `Preconditions`. If you use those, you may not need a custom `Guard` class — document which utility you standardize on.
