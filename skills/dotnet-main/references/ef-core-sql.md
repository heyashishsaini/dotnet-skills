# EF Core, SQL Server & T-SQL Reference

## DbContext Setup
```csharp
public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    public DbSet<Order> Orders => Set<Order>();
    public DbSet<Product> Products => Set<Product>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(Assembly.GetExecutingAssembly());
    }
}
```

## Entity Configuration (Fluent API — preferred over annotations)
```csharp
public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.HasKey(o => o.Id);
        builder.Property(o => o.TotalAmount).HasPrecision(18, 2).IsRequired();
        builder.HasMany(o => o.Items).WithOne(i => i.Order).HasForeignKey(i => i.OrderId);
        builder.HasIndex(o => o.CustomerId);
    }
}
```

## Repository Pattern
```csharp
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task<IReadOnlyList<Order>> GetByCustomerAsync(Guid customerId, CancellationToken ct = default);
    void Add(Order order);
}

public class OrderRepository : IOrderRepository
{
    private readonly AppDbContext _context;
    public OrderRepository(AppDbContext context) => _context = context;

    public async Task<Order?> GetByIdAsync(Guid id, CancellationToken ct)
        => await _context.Orders
            .Include(o => o.Items)
            .FirstOrDefaultAsync(o => o.Id == id, ct);
}
```

## Unit of Work
```csharp
public interface IUnitOfWork
{
    IOrderRepository Orders { get; }
    Task<int> SaveChangesAsync(CancellationToken ct = default);
}
```

## N+1 Problem (critical!)
```csharp
// ❌ N+1: separate query per order's items
var orders = await _context.Orders.ToListAsync();
foreach (var order in orders)
    Console.WriteLine(order.Items.Count); // triggers N queries

// ✅ Eager loading
var orders = await _context.Orders.Include(o => o.Items).ToListAsync();
```

## Migrations
```bash
dotnet ef migrations add InitialCreate --project Infrastructure --startup-project API
dotnet ef database update --project Infrastructure --startup-project API
```

## Raw SQL (when LINQ isn't enough)
```csharp
var orders = await _context.Orders
    .FromSqlRaw("SELECT * FROM Orders WHERE CustomerId = {0}", customerId)
    .ToListAsync();
```

## T-SQL Patterns
```sql
-- Pagination
SELECT * FROM Orders
ORDER BY CreatedAt DESC
OFFSET (@page - 1) * @pageSize ROWS
FETCH NEXT @pageSize ROWS ONLY;

-- Indexed views for reporting
CREATE INDEX IX_Orders_CustomerId ON Orders(CustomerId) INCLUDE (TotalAmount, Status);

-- Stored procedure for complex logic
CREATE PROCEDURE GetOrderSummary @CustomerId UNIQUEIDENTIFIER
AS BEGIN
    SELECT o.Id, COUNT(i.Id) AS ItemCount, SUM(i.Price) AS Total
    FROM Orders o JOIN OrderItems i ON i.OrderId = o.Id
    WHERE o.CustomerId = @CustomerId
    GROUP BY o.Id;
END
```

## Common Mistakes
1. Tracking entities when only reading (`AsNoTracking()` for reads)
2. Loading entire entity when only needing fields (use `Select()` projections)
3. Not indexing foreign keys
4. Running migrations in production without a backup
5. Using `SaveChanges()` inside a loop instead of batching
