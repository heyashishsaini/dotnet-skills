# Clean Architecture, CQRS & DI Reference

## Clean Architecture Layers
```
┌─────────────────────────────────┐
│           Presentation          │  ASP.NET Core, Controllers, Minimal APIs
├─────────────────────────────────┤
│          Infrastructure         │  EF Core, SQL Server, Email, Azure, 3rd party
├─────────────────────────────────┤
│           Application           │  Use cases, Commands, Queries, DTOs, Interfaces
├─────────────────────────────────┤
│             Domain              │  Entities, Value Objects, Domain Events, Rules
└─────────────────────────────────┘
            ↑ Dependency Rule: outer layers depend on inner layers only
```

## Project Structure
```
MyApp/
├── MyApp.Domain/           # No dependencies
├── MyApp.Application/      # Depends on Domain only
├── MyApp.Infrastructure/   # Depends on Application + Domain
└── MyApp.API/              # Depends on all (composition root)
```

## CQRS with MediatR
```csharp
// Command
public record CreateOrderCommand(Guid CustomerId, List<OrderItemDto> Items) 
    : IRequest<Guid>;

// Handler
public class CreateOrderCommandHandler : IRequestHandler<CreateOrderCommand, Guid>
{
    private readonly IUnitOfWork _uow;
    public CreateOrderCommandHandler(IUnitOfWork uow) => _uow = uow;

    public async Task<Guid> Handle(CreateOrderCommand cmd, CancellationToken ct)
    {
        var order = Order.Create(cmd.CustomerId, cmd.Items);
        _uow.Orders.Add(order);
        await _uow.SaveChangesAsync(ct);
        return order.Id;
    }
}

// Controller calls MediatR
[HttpPost]
public async Task<IActionResult> Create(CreateOrderCommand cmd)
    => Ok(await _mediator.Send(cmd));
```

## Dependency Injection — Lifetimes
| Lifetime | When to use | Example |
|---|---|---|
| Singleton | Shared state, once per app | IConfiguration, Cache |
| Scoped | Per HTTP request | DbContext, Repositories |
| Transient | Lightweight, no state | Validators, Helpers |

```csharp
// ❌ Captive dependency: Singleton holding Scoped → memory leak
services.AddSingleton<IMyService, MyService>(); // MyService holds DbContext!

// ✅ Use factory or avoid injecting scoped into singleton
```

## Domain Entity (rich model, not anemic)
```csharp
public class Order
{
    private readonly List<OrderItem> _items = new();
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();

    private Order() { } // EF constructor

    public static Order Create(Guid customerId, List<OrderItemDto> items)
    {
        if (!items.Any()) throw new DomainException("Order must have items");
        var order = new Order { Id = Guid.NewGuid(), CustomerId = customerId };
        foreach (var item in items) order._items.Add(OrderItem.Create(item));
        return order;
    }
}
```

## Common Mistakes
1. Putting business logic in controllers or repositories
2. Anemic domain models (entities are just bags of properties)
3. Referencing Infrastructure from Domain
4. Using static methods instead of DI
5. Skipping Application layer — going Controller → Repository directly
