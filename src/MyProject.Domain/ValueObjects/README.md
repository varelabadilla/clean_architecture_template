# Domain / ValueObjects

## Why This Folder Exists

Value objects model **domain concepts that have no identity** — they are defined entirely by their attributes. Two `Money` instances with `Amount: 100` and `Currency: "USD"` are the same value, regardless of when or where they were created. If any attribute changes, you don't update a value object — you replace it with a new one.

Value objects make domain code more expressive and eliminate primitive obsession (using raw `string`, `decimal`, `int` to represent rich concepts like monetary amounts, addresses, or email addresses).

---

## What Belongs Here

- Immutable types that model domain concepts without identity
- Types that replace primitive types in entity properties
- Types with domain-specific validation in their constructor
- Types with domain-specific operations and arithmetic

---

## What Does NOT Belong Here

- Types with identity (those are entities)
- Mutable types
- DTO or API contract types
- Simple enums (those go in `Enums/`)

---

## Recommended Naming Conventions

| Type | Convention | Example |
| --- | --- | --- |
| Value object class | Noun describing the concept | `Money`, `Address`, `EmailAddress`, `PhoneNumber` |
| Constructor | Private or internal, factory method preferred | `Money.Of(100, "USD")` |
| Equality | Structural (by attributes), not referential | Implemented via `record` or `Equals` override |

---

## Implementation Patterns

### Using C# records (preferred for simple value objects)

```csharp
public record Money(decimal Amount, string Currency)
{
    // Validation in constructor
    public Money
    {
        Guard.AgainstNegative((int)Amount, nameof(Amount));
        Guard.AgainstNullOrEmpty(Currency, nameof(Currency));

        if (Currency.Length != 3)
            throw new ArgumentException("Currency must be a 3-letter ISO code.", nameof(Currency));

        Currency = Currency.ToUpperInvariant();
    }

    public static Money Zero(string currency) => new(0, currency);

    public static Money operator +(Money a, Money b)
    {
        if (a.Currency != b.Currency)
            throw new CurrencyMismatchException(a.Currency, b.Currency);
        return new Money(a.Amount + b.Amount, a.Currency);
    }

    public static Money operator *(Money money, int factor) =>
        new(money.Amount * factor, money.Currency);

    public override string ToString() => $"{Amount:N2} {Currency}";
}
```

### Using abstract base class (for complex scenarios)

```csharp
public abstract class ValueObject
{
    protected abstract IEnumerable<object?> GetEqualityComponents();

    public override bool Equals(object? obj)
    {
        if (obj is null || obj.GetType() != GetType()) return false;
        return GetEqualityComponents()
            .SequenceEqual(((ValueObject)obj).GetEqualityComponents());
    }

    public override int GetHashCode() =>
        GetEqualityComponents()
            .Aggregate(0, HashCode.Combine);
}

public class Address : ValueObject
{
    public string Street { get; }
    public string City { get; }
    public string Country { get; }
    public string PostalCode { get; }

    public Address(string street, string city, string country, string postalCode)
    {
        Street = Guard.AgainstNullOrWhiteSpace(street, nameof(street));
        City = Guard.AgainstNullOrWhiteSpace(city, nameof(city));
        Country = Guard.AgainstNullOrWhiteSpace(country, nameof(country));
        PostalCode = Guard.AgainstNullOrWhiteSpace(postalCode, nameof(postalCode));
    }

    protected override IEnumerable<object?> GetEqualityComponents()
    {
        yield return Street.ToLowerInvariant();
        yield return City.ToLowerInvariant();
        yield return Country.ToUpperInvariant();
        yield return PostalCode;
    }
}
```

### Java example

```java
public final class Money {
    private final BigDecimal amount;
    private final String currency;

    private Money(BigDecimal amount, String currency) {
        if (amount.compareTo(BigDecimal.ZERO) < 0)
            throw new IllegalArgumentException("Amount cannot be negative");
        this.amount = amount;
        this.currency = currency.toUpperCase();
    }

    public static Money of(BigDecimal amount, String currency) {
        return new Money(amount, currency);
    }

    public Money add(Money other) {
        if (!this.currency.equals(other.currency))
            throw new CurrencyMismatchException(this.currency, other.currency);
        return new Money(this.amount.add(other.amount), this.currency);
    }

    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Money m)) return false;
        return amount.compareTo(m.amount) == 0 && currency.equals(m.currency);
    }

    @Override
    public int hashCode() {
        return Objects.hash(amount.stripTrailingZeros(), currency);
    }
}
```

---

## Common Value Object Examples

| Concept | Type name | Key attributes |
| --- | --- | --- |
| Monetary amount | `Money` | `Amount`, `Currency` |
| Physical address | `Address` | `Street`, `City`, `Country`, `PostalCode` |
| Email | `EmailAddress` | `Value` (validated format) |
| Phone | `PhoneNumber` | `Value` (normalized) |
| Date range | `DateRange` | `Start`, `End` (with `Contains`, `Overlaps`) |
| Geographic point | `Coordinates` | `Latitude`, `Longitude` |
| Percentage | `Percentage` | `Value` (0-100 range enforced) |
| Quantity | `Quantity` | `Amount`, `Unit` |

---

## Important Notes

- Value objects should be **immutable**. Never add setters. Return new instances from methods that transform them.
- Value objects are **replaced, not updated**. `order.UpdateShippingAddress(newAddress)` replaces the old address with a new one — it does not mutate the existing `Address` object.
- ORM persistence of value objects (owned entity types in EF Core, `@Embeddable` in JPA) is an infrastructure concern. The value object itself has no knowledge of how it is stored.
- Prefer `record` in C# for simple value objects — it gives you structural equality and immutability for free. Use the abstract `ValueObject` base class for complex cases where you need custom equality logic.
