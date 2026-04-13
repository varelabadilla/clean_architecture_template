# SharedKernel / Extensions

## Why This Folder Exists

Extension methods that operate on primitive types or standard library types — and have **no dependency on any project-specific or framework-specific code** — need a home that is accessible to all layers without creating coupling.

This folder provides that home. The constraint is strict: every method here must be self-contained and depend only on the base runtime.

---

## What Belongs Here

- Extension methods on `string`, `DateTime`, `DateTimeOffset`, `Guid`
- Extension methods on `IEnumerable<T>` or `ICollection<T>` using only BCL types
- Extension methods on `bool`, numeric types, or other primitives
- Pure transformation utilities with no side effects

---

## What Does NOT Belong Here

- Extension methods that reference Domain types
- Extension methods that reference Application or Infrastructure types
- Anything requiring a NuGet package
- Mapping helpers (those belong near the layer that uses them)

---

## Recommended File Names

```plaintext
StringExtensions.cs
DateTimeExtensions.cs
EnumerableExtensions.cs
GuidExtensions.cs
QueryableExtensions.cs    // only if using IQueryable<T> from base BCL
```

---

## Examples

```csharp
// StringExtensions.cs
public static class StringExtensions
{
    public static string ToSnakeCase(this string value) { ... }
    public static bool IsNullOrEmpty(this string? value) => string.IsNullOrEmpty(value);
    public static string Truncate(this string value, int maxLength) { ... }
}

// DateTimeExtensions.cs
public static class DateTimeExtensions
{
    public static bool IsWeekend(this DateTime date) =>
        date.DayOfWeek is DayOfWeek.Saturday or DayOfWeek.Sunday;

    public static DateTime StartOfDay(this DateTime date) =>
        date.Date;

    public static DateTime EndOfDay(this DateTime date) =>
        date.Date.AddDays(1).AddTicks(-1);
}

// EnumerableExtensions.cs
public static class EnumerableExtensions
{
    public static bool IsNullOrEmpty<T>(this IEnumerable<T>? source) =>
        source is null || !source.Any();

    public static IEnumerable<IEnumerable<T>> Batch<T>(
        this IEnumerable<T> source, int size) { ... }
}
```

---

## Naming Conventions

- File name: `{TypeBeingExtended}Extensions.cs`
- Class name matches file name
- Methods use clear verb phrases: `ToSnakeCase`, `IsNullOrEmpty`, `StartOfDay`

---

## Important Notes

- Keep methods short and pure. If a method needs more than ~20 lines, it likely belongs in a dedicated utility class rather than an extension method.
- In Java, extension methods do not exist as a language feature. Equivalent functionality lives in `static` utility classes (e.g., `StringUtils`, `DateUtils`). Apply the same constraints: no framework or project dependencies.
- Review this folder periodically. It is easy for it to grow into a grab-bag of unrelated utilities. If a group of methods forms a coherent concept, consider whether they deserve their own home.
