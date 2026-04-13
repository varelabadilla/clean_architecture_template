# Application / Abstractions

## Why This Folder Exists

The Application layer needs to interact with external services — sending emails, uploading files, processing payments, publishing messages — but it cannot depend on any concrete implementation of those services without coupling itself to infrastructure.

This folder solves that by defining **the interfaces that external services must implement**. The Application layer depends only on these interfaces. Infrastructure implements them. This is the Dependency Inversion Principle in practice.

---

## What Belongs Here

- Interfaces for external services the Application layer uses
- Interfaces for cross-cutting infrastructure services (current user, date/time provider)
- No implementations — only contracts

---

## What Does NOT Belong Here

- Domain repository interfaces (those go in `Domain/Repositories/`)
- Concrete service implementations (those go in `Infrastructure/Services/`)
- DTOs or response models
- Anything framework-specific

---

## Recommended Naming Conventions

| Type | Convention | Example |
| --- | --- | --- |
| Service interface | `I` + service noun | `IEmailService`, `IFileStorageService` |
| Context interface | `I` + context noun | `ICurrentUserContext`, `IDateTimeProvider` |
| Methods | Verb phrase expressing the operation | `SendAsync`, `UploadAsync`, `GetCurrentUserId` |

---

## Common Interfaces

### IEmailService

```csharp
public interface IEmailService
{
    Task SendAsync(
        string to,
        string subject,
        string body,
        bool isHtml = true,
        CancellationToken cancellationToken = default);

    Task SendTemplatedAsync(
        string to,
        string templateId,
        object templateData,
        CancellationToken cancellationToken = default);
}
```

### IFileStorageService

```csharp
public interface IFileStorageService
{
    Task<string> UploadAsync(
        Stream fileStream,
        string fileName,
        string contentType,
        CancellationToken cancellationToken = default);

    Task<Stream> DownloadAsync(
        string fileKey,
        CancellationToken cancellationToken = default);

    Task DeleteAsync(
        string fileKey,
        CancellationToken cancellationToken = default);
}
```

### ICurrentUserContext

```csharp
public interface ICurrentUserContext
{
    Guid UserId { get; }
    string Email { get; }
    IReadOnlyList<string> Roles { get; }
    bool IsAuthenticated { get; }
}
```

### IDateTimeProvider

```csharp
// Abstracting DateTime.UtcNow makes time-dependent logic fully testable
public interface IDateTimeProvider
{
    DateTime UtcNow { get; }
    DateOnly Today { get; }
}
```

### ICacheService

```csharp
public interface ICacheService
{
    Task<T?> GetAsync<T>(string key, CancellationToken cancellationToken = default);

    Task SetAsync<T>(
        string key,
        T value,
        TimeSpan? expiry = null,
        CancellationToken cancellationToken = default);

    Task RemoveAsync(string key, CancellationToken cancellationToken = default);
}
```

### IMessagePublisher

```csharp
public interface IMessagePublisher
{
    Task PublishAsync<T>(T message, CancellationToken cancellationToken = default)
        where T : class;
}
```

---

## Java Equivalent

```java
// Each abstraction is a plain Java interface in the application package
public interface EmailService {
    void send(String to, String subject, String body);
    void sendTemplated(String to, String templateId, Map<String, Object> data);
}

public interface CurrentUserContext {
    UUID getUserId();
    String getEmail();
    List<String> getRoles();
    boolean isAuthenticated();
}

public interface DateTimeProvider {
    LocalDateTime utcNow();
    LocalDate today();
}
```

---

## Why IDateTimeProvider Matters

Never use `DateTime.UtcNow` directly in Application handlers or Domain logic. It makes time-dependent behavior impossible to test deterministically.

```csharp
// Hard to test — what time is "now" in the test?
if (order.CreatedAt < DateTime.UtcNow.AddDays(-30))

// Testable — inject a mock IDateTimeProvider
if (order.CreatedAt < _dateTimeProvider.UtcNow.AddDays(-30))
```

---

## Important Notes

- Keep these interfaces **focused**. An `IEmailService` with 15 methods is a sign that responsibilities have been mixed. Split by concern when needed.
- Every interface here is a **testability seam**. In unit tests, you replace implementations with mocks or fakes. Well-designed interfaces make this trivial.
- Do not add optional parameters or overloads excessively. Lean interfaces are easier to mock and implement.
- In Spring Boot, these interfaces are commonly annotated with `@Service` on their implementations. The interface itself has no Spring annotations — it is a pure Java interface.
