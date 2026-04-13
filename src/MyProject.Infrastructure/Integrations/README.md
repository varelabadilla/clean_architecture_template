# Infrastructure / Integrations

## Why This Folder Exists

This folder contains **clients and adapters for external third-party systems**: payment processors, SMS gateways, object storage providers, external APIs, SMTP servers, and similar services. Each integration is isolated in its own subfolder to contain its dependencies and configuration.

The distinction from `Services/` is scope: `Services/` contains implementations of internal application abstractions (like `ICurrentUserContext`). `Integrations/` contains wrappers around external providers that may need API keys, webhooks, or SDKs.

---

## What Belongs Here

- Payment gateway adapters (Stripe, PayPal, Braintree)
- Email provider clients (SendGrid, Mailgun, AWS SES)
- SMS providers (Twilio, Vonage)
- Object storage adapters (AWS S3, Azure Blob Storage, GCS)
- Third-party API clients and their DTOs
- Webhook handlers for incoming external events

---

## What Does NOT Belong Here

- Interfaces (those live in `Application/Abstractions/`)
- Business logic
- Internal service implementations that don't talk to an external system

---

## Recommended Structure

Organize by provider, not by function:

```plaintext
Integrations/
├─ Stripe/
│  ├─ StripePaymentAdapter.cs       // implements IPaymentService
│  ├─ StripeWebhookHandler.cs
│  ├─ StripeOptions.cs              // configuration binding
│  └─ Models/                       // Stripe-specific DTOs
│     ├─ StripeChargeResponse.cs
│     └─ StripeWebhookEvent.cs
│
├─ SendGrid/
│  ├─ SendGridEmailService.cs       // implements IEmailService
│  ├─ SendGridOptions.cs
│  └─ Templates/
│
├─ AWSS3/
│  ├─ S3FileStorageService.cs       // implements IFileStorageService
│  └─ S3Options.cs
│
└─ Twilio/
   ├─ TwilioSmsService.cs           // implements ISmsService
   └─ TwilioOptions.cs
```

---

## Naming Conventions

| Type | Convention | Example |
| --- | --- | --- |
| Adapter class | ProviderName + ConceptName | `StripePaymentAdapter`, `SendGridEmailService` |
| Options class | ProviderName + `Options` | `StripeOptions`, `TwilioOptions` |
| Provider-specific DTO | ProviderName + concept | `StripeChargeResponse` |

---

## Adapter Example

```csharp
public class StripePaymentAdapter : IPaymentService
{
    private readonly StripeClient _client;
    private readonly StripeOptions _options;
    private readonly ILogger<StripePaymentAdapter> _logger;

    public StripePaymentAdapter(
        StripeClient client,
        IOptions<StripeOptions> options,
        ILogger<StripePaymentAdapter> logger)
    {
        _client = client;
        _options = options.Value;
        _logger = logger;
    }

    public async Task<Result<PaymentConfirmation>> ChargeAsync(
        decimal amount, string currency, string paymentMethodId,
        CancellationToken cancellationToken = default)
    {
        try
        {
            var options = new PaymentIntentCreateOptions
            {
                Amount = (long)(amount * 100),
                Currency = currency.ToLower(),
                PaymentMethod = paymentMethodId,
                Confirm = true,
            };

            var service = new PaymentIntentService(_client);
            var intent = await service.CreateAsync(options, cancellationToken: cancellationToken);

            return Result<PaymentConfirmation>.Success(
                new PaymentConfirmation(intent.Id, intent.Status));
        }
        catch (StripeException ex)
        {
            _logger.LogError(ex, "Stripe payment failed for amount {Amount} {Currency}", amount, currency);
            return Result<PaymentConfirmation>.Failure(
                Error.Failure("Payment.Failed", ex.StripeError?.Message ?? ex.Message));
        }
    }
}
```

---

## Configuration Options Pattern

Each integration should use strongly-typed options:

```csharp
public class StripeOptions
{
    public const string SectionName = "Stripe";
    public string SecretKey { get; init; } = string.Empty;
    public string WebhookSecret { get; init; } = string.Empty;
    public string PublishableKey { get; init; } = string.Empty;
}

// Registered in DependencyInjection.cs:
services.Configure<StripeOptions>(
    configuration.GetSection(StripeOptions.SectionName));
```

```json
// appsettings.json
{
  "Stripe": {
    "SecretKey": "sk_test_...",
    "WebhookSecret": "whsec_..."
  }
}
```

---

## Important Notes

- Each integration adapter catches provider-specific exceptions and translates them into `Result<T>` failures or application exceptions. Provider exceptions (like `StripeException`) should never propagate beyond this folder.
- API keys and secrets must never be hardcoded. Use environment variables, Azure Key Vault, AWS Secrets Manager, or similar. The options class just provides the binding shape.
- In Java/Spring, each integration is typically a `@Service`-annotated adapter. Use `@ConfigurationProperties` for type-safe configuration binding.
- Integration tests for this folder run against sandboxed/test accounts of the external services, or use recorded HTTP responses with tools like WireMock.
