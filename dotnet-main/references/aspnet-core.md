# ASP.NET Core & Web API Reference

## Request Pipeline
```
Request → Kestrel → Middleware Pipeline → Routing → Controller → Action → Response
```

## Middleware Order (matters!)
```csharp
app.UseExceptionHandler();   // 1. Catch all errors first
app.UseHttpsRedirection();   // 2. Force HTTPS
app.UseStaticFiles();        // 3. Serve static files early
app.UseRouting();            // 4. Match routes
app.UseAuthentication();     // 5. Who are you?
app.UseAuthorization();      // 6. What can you do?
app.UseRateLimiter();        // 7. Throttle requests
app.MapControllers();        // 8. Execute controller
```

## Controller Best Practices
```csharp
[ApiController]
[Route("api/v1/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly IOrderService _orderService;
    private readonly ILogger<OrdersController> _logger;

    public OrdersController(IOrderService orderService, ILogger<OrdersController> logger)
    {
        _orderService = orderService;
        _logger = logger;
    }

    [HttpGet("{id:guid}")]
    [ProducesResponseType(typeof(OrderDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<OrderDto>> GetOrder(Guid id, CancellationToken ct)
    {
        var order = await _orderService.GetByIdAsync(id, ct);
        return order is null ? NotFound() : Ok(order);
    }
}
```

## Global Exception Handling
```csharp
// Program.cs
app.UseExceptionHandler(appError =>
{
    appError.Run(async context =>
    {
        context.Response.StatusCode = 500;
        context.Response.ContentType = "application/json";
        var error = context.Features.Get<IExceptionHandlerFeature>();
        // log + write ProblemDetails response
    });
});
```

## Custom Middleware
```csharp
public class RequestLoggingMiddleware(RequestDelegate next, ILogger<RequestLoggingMiddleware> logger)
{
    public async Task InvokeAsync(HttpContext context)
    {
        logger.LogInformation("Request: {Method} {Path}", context.Request.Method, context.Request.Path);
        await next(context);
        logger.LogInformation("Response: {StatusCode}", context.Response.StatusCode);
    }
}
```

## Minimal API vs Controller API
| | Controller | Minimal API |
|---|---|---|
| Use when | Large APIs, teams | Small APIs, microservices |
| Swagger | Auto | Manual |
| Filters | Built-in | Manual middleware |
| Testability | Easy | Slightly harder |

## API Versioning
```csharp
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
});
```

## Common Mistakes
1. Putting business logic in controllers
2. Not using `[ApiController]` attribute
3. Returning raw exceptions to clients
4. Wrong middleware order (auth before routing)
5. Not cancelling async operations with CancellationToken
