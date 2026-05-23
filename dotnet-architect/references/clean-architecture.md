# Clean Architecture & CQRS Patterns in .NET

## When to Use Clean Architecture

Clean Architecture pays off when:
- The domain has real complexity (business rules, not just CRUD)
- Multiple teams work on the codebase
- Long-term maintainability matters more than speed of first delivery
- You anticipate changing infrastructure (swap SQL Server for PostgreSQL, etc.)

It adds overhead that hurts on simple CRUD apps. For small projects, a simpler layered architecture
(Controllers → Services → Repositories → DbContext) is often the right call. Don't let Clean
Architecture become a cargo cult.

## Layer Responsibilities

### Domain Layer
- No dependencies on any other layer or NuGet package (except primitives)
- Contains: Entities, Value Objects, Domain Events, Enums, Domain Exceptions, Repository Interfaces
- Entities enforce their own invariants — they don't have public setters everywhere
- Value Objects are immutable and compared by value, not reference

```csharp
// Good domain entity — protects its own state
public class Order
{
    private readonly List<OrderItem> _items = new();
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();
    public OrderStatus Status { get; private set; }

    public void AddItem(Product product, int quantity)
    {
        if (Status != OrderStatus.Draft)
            throw new DomainException("Cannot modify a submitted order.");
        _items.Add(new OrderItem(product.Id, product.Price, quantity));
    }
}
```

### Application Layer
- Depends only on Domain
- Contains: Commands, Queries, Handlers, DTOs, Validators, Application Exceptions
- Orchestrates domain objects — no business logic lives here
- Uses MediatR for command/query dispatch

```csharp
public record CreateOrderCommand(Guid CustomerId, List<OrderItemDto> Items) : IRequest<Guid>;

public class CreateOrderCommandHandler : IRequestHandler<CreateOrderCommand, Guid>
{
    private readonly IOrderRepository _orders;
    private readonly IUnitOfWork _uow;

    public async Task<Guid> Handle(CreateOrderCommand request, CancellationToken ct)
    {
        var order = Order.Create(request.CustomerId);
        foreach (var item in request.Items)
            order.AddItem(...);
        _orders.Add(order);
        await _uow.SaveChangesAsync(ct);
        return order.Id;
    }
}
```

### Infrastructure Layer
- Implements interfaces defined in Domain/Application
- Contains: EF Core DbContext, Repository implementations, external HTTP clients, email, Redis, etc.
- This is where all the messy I/O lives — keep it isolated

### Presentation Layer (API)
- Thin controllers — just parse the request, dispatch to MediatR, return the response
- No business logic. No direct DB access. No domain objects in responses.

```csharp
[HttpPost]
public async Task<IActionResult> CreateOrder(CreateOrderRequest request, CancellationToken ct)
{
    var command = new CreateOrderCommand(request.CustomerId, request.Items);
    var orderId = await _mediator.Send(command, ct);
    return CreatedAtAction(nameof(GetOrder), new { id = orderId }, null);
}
```

## CQRS Guidance

Use CQRS (Command Query Responsibility Segregation) when:
- Read models differ significantly from write models
- Read performance needs optimization independently
- You have complex reporting queries that shouldn't go through the domain

Don't use CQRS just because you're using MediatR. MediatR is a mediator pattern library —
it doesn't require CQRS. A simple app can use MediatR with a unified model.

CQRS maturity levels:
1. **Basic** — Same DB, separate command/query handlers (MediatR)
2. **Optimized reads** — Raw SQL / Dapper for queries, EF Core for commands
3. **Separate read store** — Redis or a read-optimized DB for projections
4. **Event sourcing** — Almost never warranted unless you need full audit trail as a first-class concern

## Pipeline Behaviors (MediatR)

Use pipeline behaviors for cross-cutting concerns:

```csharp
// Register in order: Logging → Validation → Transaction
services.AddTransient(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
services.AddTransient(typeof(IPipelineBehavior<,>), typeof(TransactionBehavior<,>));
```

ValidationBehavior runs FluentValidation validators automatically — no need to call `.Validate()`
manually in every handler.

## Repository Pattern Guidance

Don't wrap every entity in a repository mechanically. EF Core's DbContext is already a Unit of Work
and DbSet is already a repository. Add a repository abstraction when:
- You need to swap the data store (rare but real)
- You need to unit test without hitting a database
- The query logic is complex enough to deserve a named home

For simple CRUD, querying through DbContext directly in the handler is fine.
