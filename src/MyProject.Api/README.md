# MyProject.Api

## Why This Project Exists

The Api project is the **entry point** of the application. It handles HTTP concerns: routing, model binding, authentication, authorization filters, response shaping, and error translation.

Its second and equally important role is acting as the **Composition Root** — the single place in the entire solution where all dependencies are wired together. Every `DependencyInjection` extension method from every layer is called here, and here alone.

---

## What Belongs Here

- Controllers (or Minimal API endpoint definitions)
- Middleware (global exception handling, request logging, rate limiting)
- Filters (action filters, authorization filters)
- Request/response contract models (`Contracts/`)
- Dependency injection composition (`DependencyInjection/`)
- `Program.cs` — the composition root
- Application settings (`appsettings.json`, `appsettings.*.json`)

---

## What Does NOT Belong Here

- Business logic (belongs in Domain)
- Use case orchestration (belongs in Application)
- Database or infrastructure code (belongs in Infrastructure)
- Domain entities or DTOs used directly as API response models — always use `Contracts/` types

---

## Dependency Rule

```plaintext
MyProject.Api → MyProject.Application, MyProject.Infrastructure (DI only), MyProject.SharedKernel
```

The reference to `Infrastructure` is **exclusively** for DI registration in `DependencyInjection/`. Controllers and middleware must never reference Infrastructure types directly.

---

## Program.cs — Composition Root

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddApplicationServices(builder.Configuration)
    .AddInfrastructureServices(builder.Configuration)
    .AddPresentationServices(builder.Configuration);

builder.Services.AddAuthentication(...)
    .AddJwtBearer(...);

var app = builder.Build();

app.UseExceptionHandler();
app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

`Program.cs` should read like a table of contents — it calls named extension methods and configures middleware. It contains no implementation details.

---

## Recommended Naming Conventions

| Type | Convention | Example |
| --- | --- | --- |
| Controller | Resource noun (plural) + `Controller` | `OrdersController`, `CustomersController` |
| Endpoint group (Minimal API) | Resource noun + `Endpoints` | `OrderEndpoints` |
| Request contract | Action + resource + `Request` | `PlaceOrderRequest`, `UpdateCustomerRequest` |
| Response contract | Resource + detail level + `Response` | `OrderResponse`, `OrderSummaryResponse` |
| Middleware | Descriptive + `Middleware` | `GlobalExceptionMiddleware`, `RequestLoggingMiddleware` |
| Filter | Descriptive + `Filter` | `IdempotencyFilter`, `RateLimitFilter` |

---

## Folder Overview

| Folder/File | Contents |
| --- | --- |
| `Controllers/` | MVC controllers or Minimal API endpoint classes |
| `Middleware/` | HTTP middleware pipeline components |
| `Filters/` | Action filters and authorization filters |
| `Contracts/` | HTTP request and response models |
| `DependencyInjection/` | Layer-specific extension methods + Composition Root support |
| `Program.cs` | Application bootstrap and composition root |
| `appsettings.json` | Base configuration |
| `appsettings.Development.json` | Development overrides |

---

## Java/Spring Equivalent

In Spring Boot, this layer maps to:

- `@RestController` classes in a `web` or `api` package
- `@ControllerAdvice` for global exception handling
- `WebMvcConfigurer` or `SecurityFilterChain` for filter configuration
- `application.yml` / `application-{profile}.yml` for configuration
- A main `@SpringBootApplication` class as the entry point
