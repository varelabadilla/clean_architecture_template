# Features / Orders / Validators

## Purpose

Contains **FluentValidation validator classes** for Order commands. Validators enforce input correctness *before* the handler runs — they answer the question "is this request well-formed and complete?" not "is this request business-valid?"

## Naming Convention

Command or query name + `Validator`:

```plaintext
PlaceOrderCommandValidator
CancelOrderCommandValidator
UpdateShippingAddressCommandValidator
```

## Rules

- One validator per command (queries rarely need validators unless they have complex filter constraints)
- Validators check **input shape**: required fields, value ranges, string lengths, format constraints
- Validators do **NOT** check **business rules** — those belong in domain entities
- Validators are auto-discovered and executed by `ValidationBehavior` — no manual wiring needed
- Validation failures short-circuit the pipeline: the handler is never called if validation fails

## Validation vs Domain Rules — The Key Distinction

| What to validate here (input) | What belongs in Domain |
| --- | --- |
| `CustomerId` must not be `Guid.Empty` | Customer must exist in the database |
| `Quantity` must be > 0 | Product must have sufficient stock |
| `ShippingAddress` fields must not be empty | Order must be in `Pending` state to add items |
| Items list must not be empty | Each product must be active and purchasable |

If a rule requires calling a repository or checking domain state — it belongs in the handler or domain, not here.

## Example

```csharp
public class PlaceOrderCommandValidator : AbstractValidator<PlaceOrderCommand>
{
    public PlaceOrderCommandValidator()
    {
        RuleFor(x => x.CustomerId)
            .NotEmpty().WithMessage("Customer ID is required.");

        RuleFor(x => x.Items)
            .NotEmpty().WithMessage("Order must contain at least one item.")
            .Must(items => items.Count <= 50)
            .WithMessage("Order cannot contain more than 50 items.");

        RuleForEach(x => x.Items).ChildRules(item =>
        {
            item.RuleFor(i => i.ProductId).NotEmpty();
            item.RuleFor(i => i.Quantity).GreaterThan(0)
                .WithMessage("Quantity must be greater than zero.");
        });

        RuleFor(x => x.ShippingAddress).NotNull()
            .WithMessage("Shipping address is required.");
    }
}
```

## Example files

```plaintext
PlaceOrderCommandValidator.cs
CancelOrderCommandValidator.cs
UpdateShippingAddressCommandValidator.cs
```
