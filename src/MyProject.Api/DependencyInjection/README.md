# Api / DependencyInjection

## Why This Folder Exists

This folder is the **Composition Root coordinator** for the entire solution. It contains the extension methods that wire together all layers and expose a clean, readable registration surface to `Program.cs`.

The key architectural decision here is explicit: the Api project is the only place that knows about all layers simultaneously. This folder makes that wiring visible, organized, and traceable — instead of burying it inside a bloated `Program.cs`.

---

## What Belongs Here

- `ApplicationServiceExtensions.cs` — delegates to `Application/DependencyInjection.cs`
- `InfrastructureServiceExtensions.cs` — delegates to `Infrastructure/DependencyInjection.cs`
- `PresentationServiceExtensions.cs` — registers API-level services: controllers, Swagger, filters, CORS, rate limiting

---

## What Does NOT Belong Here

- Actual service registration logic (each layer owns its own registrations)
- Business logic
- Middleware registration (that belongs in `Program.cs` pipeline configuration)

---

## File Responsibilities

### ApplicationServiceExtensions.cs

A thin pass-through that calls the Application layer's own registration method:

```csharp
public static class ApplicationServiceExtensions
{
    public static IServiceCollection AddApplicationServices(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        // Delegates entirely to the Application layer's own DI method
        return services.AddApplicationServices(configuration);
        // Note: if namespace conflict, use full type name or alias
    }
}
```

> In practice, since `Application/DependencyInjection.cs` already defines `AddApplicationServices()`, you may call it directly from `Program.cs` without a pass-through. Use this file when you need to add API-level configuration around the Application layer's registration — for example, registering additional MediatR behaviors that are specific to the HTTP context.

---

### InfrastructureServiceExtensions.cs

Same pattern — delegates to Infrastructure and adds any API-level infrastructure concerns:

```csharp
public static class InfrastructureServiceExtensions
{
    public static IServiceCollection AddInfrastructureServices(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddInfrastructureServices(configuration); // from Infrastructure layer

        // API-specific infrastructure additions
        services.AddHttpContextAccessor();

        return services;
    }
}
```

---

### PresentationServiceExtensions.cs

Registers everything that is purely an HTTP presentation concern:

```csharp
public static class PresentationServiceExtensions
{
    public static IServiceCollection AddPresentationServices(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddControllers(options =>
        {
            options.Filters.Add<IdempotencyFilter>();
        })
        .AddJsonOptions(options =>
        {
            options.JsonSerializerOptions.Converters.Add(new JsonStringEnumConverter());
            options.JsonSerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
        });

        services.AddEndpointsApiExplorer();
        services.AddSwaggerGen(options =>
        {
            options.SwaggerDoc("v1", new OpenApiInfo
            {
                Title = "MyProject API",
                Version = "v1"
            });
            options.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
            {
                Type = SecuritySchemeType.Http,
                Scheme = "bearer",
                BearerFormat = "JWT"
            });
        });

        services.AddCors(options =>
        {
            options.AddPolicy("AllowFrontend", policy =>
                policy
                    .WithOrigins(configuration["Cors:AllowedOrigins"]?.Split(',') ?? [])
                    .AllowAnyHeader()
                    .AllowAnyMethod());
        });

        services.AddRateLimiter(options =>
        {
            options.AddFixedWindowLimiter("api", limiter =>
            {
                limiter.Window = TimeSpan.FromMinutes(1);
                limiter.PermitLimit = 100;
            });
        });

        return services;
    }
}
```

---

## Program.cs — The Result

With these three extension methods, `Program.cs` becomes a clean table of contents:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddApplicationServices(builder.Configuration)
    .AddInfrastructureServices(builder.Configuration)
    .AddPresentationServices(builder.Configuration);

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseGlobalExceptionHandler();
app.UseHttpsRedirection();
app.UseCors("AllowFrontend");
app.UseAuthentication();
app.UseAuthorization();
app.UseRateLimiter();
app.MapControllers();

app.Run();
```

No implementation details. No magic. Every line is either "register services" or "configure pipeline."

---

## Important Notes

- If a registration appears in this folder, it is a presentation concern. If it belongs to another layer, move it to that layer's own `DependencyInjection.cs`.
- The three-file split (`Application`, `Infrastructure`, `Presentation`) maps directly to the three layers the Api project wires together. If you add a new cross-cutting concern, decide which layer it belongs to before placing it here.
- In Java/Spring Boot, there is no direct equivalent because Spring uses component scanning and `@Configuration` classes. The closest analogy is having separate `@Configuration` classes per layer, all loaded by the main `@SpringBootApplication` class.
