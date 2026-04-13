# Api / Controllers

## Why This Folder Exists

Controllers are the **HTTP entry points** into the application. Their responsibility is narrow: accept an HTTP request, validate the shape of the input (not business rules — that is the Application layer's job), dispatch a command or query via MediatR, and return an HTTP response.

A well-written controller is thin. If a controller method exceeds ~15 lines, business or orchestration logic has leaked into it.

---

## What Belongs Here

- `ControllerBase` subclasses with `[ApiController]` attribute
- Minimal API endpoint registration classes (if using Minimal APIs instead)
- A base `ApiController` class with shared behavior (MediatR sender injection, common responses)

---

## What Does NOT Belong Here

- Business logic
- Direct repository or service calls (always go through MediatR/CQRS)
- DTO mapping (map request contracts to commands/queries here; domain-to-DTO mapping lives in Application)
- Authentication or authorization implementation logic

---

## Recommended Naming Conventions

| Type | Convention | Example |
| --- | --- | --- |
| Controller class | PascalCase plural resource + `Controller` | `OrdersController`, `CustomersController` |
| Action method | HTTP verb or domain verb | `GetById`, `PlaceOrder`, `Cancel` |
| Route | `api/[controller]` base, specific verbs | `GET api/orders/{id}` |

---

## Base Controller

Define a shared base to avoid repeating MediatR injection and common response helpers:

```csharp
[ApiController]
[Route("api/[controller]")]
public abstract class ApiController : ControllerBase
{
    private ISender? _sender;
    protected ISender Sender =>
        _sender ??= HttpContext.RequestServices.GetRequiredService<ISender>();

    protected IActionResult HandleResult<T>(Result<T> result)
    {
        if (result.IsSuccess)
            return Ok(result.Value);

        return result.Error.Type switch
        {
            ErrorType.NotFound    => NotFound(result.Error),
            ErrorType.Validation  => UnprocessableEntity(result.Error),
            ErrorType.Forbidden   => Forbid(),
            ErrorType.Conflict    => Conflict(result.Error),
            _                     => StatusCode(500, result.Error)
        };
    }

    protected IActionResult HandleResult(Result result)
    {
        if (result.IsSuccess) return NoContent();
        return result.Error.Type switch
        {
            ErrorType.NotFound   => NotFound(result.Error),
            ErrorType.Validation => UnprocessableEntity(result.Error),
            _                    => StatusCode(500, result.Error)
        };
    }
}
```

---

## Controller Example

```csharp
public class OrdersController : ApiController
{
    [HttpGet("{id:guid}")]
    [ProducesResponseType(typeof(OrderResponse), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> GetById(
        Guid id, CancellationToken cancellationToken)
    {
        var result = await Sender.Send(new GetOrderByIdQuery(id), cancellationToken);
        return HandleResult(result);
    }

    [HttpPost]
    [Authorize]
    [ProducesResponseType(typeof(Guid), StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status422UnprocessableEntity)]
    public async Task<IActionResult> PlaceOrder(
        [FromBody] PlaceOrderRequest request,
        CancellationToken cancellationToken)
    {
        var command = new PlaceOrderCommand(
            request.CustomerId,
            request.Items.Select(i => new OrderItemRequest(i.ProductId, i.Quantity)).ToList(),
            request.ShippingAddress.ToDto());

        var result = await Sender.Send(command, cancellationToken);

        return result.IsSuccess
            ? CreatedAtAction(nameof(GetById), new { id = result.Value }, result.Value)
            : HandleResult(result);
    }

    [HttpDelete("{id:guid}")]
    [Authorize]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> Cancel(
        Guid id,
        [FromBody] CancelOrderRequest request,
        CancellationToken cancellationToken)
    {
        var result = await Sender.Send(
            new CancelOrderCommand(id, request.Reason), cancellationToken);
        return HandleResult(result);
    }
}
```

---

## Minimal APIs Alternative

If using Minimal APIs instead of controllers:

```csharp
// Api/Controllers/OrderEndpoints.cs
public static class OrderEndpoints
{
    public static IEndpointRouteBuilder MapOrderEndpoints(
        this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("api/orders")
            .WithTags("Orders")
            .RequireAuthorization();

        group.MapGet("{id:guid}", GetById);
        group.MapPost("", PlaceOrder);

        return app;
    }

    private static async Task<IResult> GetById(
        Guid id, ISender sender, CancellationToken ct)
    {
        var result = await sender.Send(new GetOrderByIdQuery(id), ct);
        return result.IsSuccess ? Results.Ok(result.Value) : Results.NotFound();
    }

    private static async Task<IResult> PlaceOrder(
        PlaceOrderRequest request, ISender sender, CancellationToken ct)
    {
        var command = request.ToCommand();
        var result = await sender.Send(command, ct);
        return result.IsSuccess
            ? Results.Created($"/api/orders/{result.Value}", result.Value)
            : Results.UnprocessableEntity(result.Error);
    }
}
```

---

## Important Notes

- Controllers only call MediatR. Never inject repositories, domain services, or infrastructure directly into a controller.
- `[ProducesResponseType]` attributes are important for Swagger/OpenAPI documentation — annotate all possible response codes.
- Map `PlaceOrderRequest` → `PlaceOrderCommand` at the controller level. The command is an application concern; the request is an HTTP contract concern. These shapes may diverge over time.
- In Java/Spring, the equivalent is `@RestController` with `@GetMapping`, `@PostMapping` etc. Use a `CommandBus` or `ApplicationEventPublisher` as the equivalent of MediatR's `ISender`.
