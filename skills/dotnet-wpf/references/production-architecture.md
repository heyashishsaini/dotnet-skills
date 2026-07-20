# WPF Production Architecture Reference

## Full Project Structure

```
src/
  MyApp.Domain/           ← Entities, interfaces, business rules (no WPF dependency)
  MyApp.Application/      ← Use cases, services, DTOs, validation
  MyApp.Infrastructure/   ← EF Core, HttpClient, file system, logging setup
  MyApp.WPF/              ← WPF project (entry point)
    App.xaml              ← App startup, global resources
    App.xaml.cs           ← DI container setup, bootstrapping
    Assets/               ← Icons, images, fonts
    Controls/             ← Reusable UserControls
    Converters/           ← IValueConverter implementations
    Resources/            ← ResourceDictionaries, themes, styles
    ViewModels/           ← One ViewModel per View
      Base/               ← ViewModelBase, RelayCommand
      Dialogs/            ← Dialog-specific ViewModels
    Views/                ← XAML views, minimal code-behind
    Services/             ← WPF-specific services (dialog service, navigation)
```

---

## DI Setup in App.xaml.cs

```csharp
public partial class App : Application
{
    public static IServiceProvider Services { get; private set; } = null!;

    protected override void OnStartup(StartupEventArgs e)
    {
        var services = new ServiceCollection();
        ConfigureServices(services);
        Services = services.BuildServiceProvider();

        var mainWindow = Services.GetRequiredService<MainWindow>();
        mainWindow.Show();
    }

    private static void ConfigureServices(IServiceCollection services)
    {
        // Infrastructure
        services.AddDbContext<AppDbContext>(options =>
            options.UseSqlServer(ConfigurationHelper.GetConnectionString()));

        services.AddHttpClient<IProductApiService, ProductApiService>(client =>
            client.BaseAddress = new Uri("https://api.example.com/"));

        // Application Services
        services.AddScoped<IProductRepository, ProductRepository>();
        services.AddScoped<IProductService, ProductService>();

        // WPF Services
        services.AddSingleton<INavigationService, NavigationService>();
        services.AddSingleton<IDialogService, DialogService>();

        // ViewModels — Transient so each navigation gets a fresh VM
        services.AddTransient<MainViewModel>();
        services.AddTransient<DashboardViewModel>();
        services.AddTransient<ProductListViewModel>();
        services.AddTransient<ProductEditViewModel>();

        // Views
        services.AddTransient<MainWindow>();
        services.AddTransient<DashboardView>();
        services.AddTransient<ProductListView>();
    }
}
```

---

## IDialogService (Testable Dialogs)

Never call `MessageBox.Show()` directly in a ViewModel. It's untestable.

```csharp
public interface IDialogService
{
    bool Confirm(string message, string title = "Confirm");
    void ShowError(string message);
    void ShowInfo(string message);
    T? ShowDialog<T>(ViewModelBase viewModel) where T : Window;
}

public class DialogService : IDialogService
{
    public bool Confirm(string message, string title = "Confirm")
        => MessageBox.Show(message, title, MessageBoxButton.YesNo) == MessageBoxResult.Yes;

    public void ShowError(string message)
        => MessageBox.Show(message, "Error", MessageBoxButton.OK, MessageBoxImage.Error);

    public void ShowInfo(string message)
        => MessageBox.Show(message, "Information", MessageBoxButton.OK, MessageBoxImage.Information);
}
```

---

## EF Core Integration

```csharp
// AppDbContext.cs in Infrastructure
public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    public DbSet<Product> Products => Set<Product>();
    public DbSet<Category> Categories => Set<Category>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }
}

// Repository in Infrastructure
public class ProductRepository : IProductRepository
{
    private readonly AppDbContext _db;
    public ProductRepository(AppDbContext db) => _db = db;

    public async Task<List<Product>> GetAllAsync(CancellationToken ct = default)
        => await _db.Products.AsNoTracking().ToListAsync(ct);

    public async Task<Product?> GetByIdAsync(int id, CancellationToken ct = default)
        => await _db.Products.FindAsync(new object[] { id }, ct);

    public async Task SaveAsync(Product product, CancellationToken ct = default)
    {
        _db.Products.Add(product);
        await _db.SaveChangesAsync(ct);
    }
}
```

**WPF-specific EF concern:** DbContext is scoped. In a desktop app, "scoped" means
per-ViewModel-lifetime or per-operation — not per HTTP request like in web apps.
Use `IDbContextFactory<AppDbContext>` for fine-grained control.

---

## HttpClient in WPF

```csharp
// Register with DI — avoids socket exhaustion
services.AddHttpClient<IProductApiService, ProductApiService>(client =>
    client.BaseAddress = new Uri("https://api.example.com/"));

// Service implementation
public class ProductApiService : IProductApiService
{
    private readonly HttpClient _client;
    public ProductApiService(HttpClient client) => _client = client;

    public async Task<List<ProductDto>> GetProductsAsync(CancellationToken ct = default)
    {
        var response = await _client.GetAsync("products", ct);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<List<ProductDto>>(cancellationToken: ct)
               ?? new List<ProductDto>();
    }
}
```

**Never** create `new HttpClient()` in a ViewModel or service. Socket exhaustion is a
real production failure mode that doesn't show up in development.

---

## Serilog Setup

```csharp
// In App.xaml.cs before DI setup
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Debug()
    .WriteTo.File("logs/app-.log", rollingInterval: RollingInterval.Day)
    .WriteTo.Debug()
    .CreateLogger();

// Register with DI
services.AddLogging(logging => logging.AddSerilog());
```

```csharp
// In ViewModel
public class ProductListViewModel : ViewModelBase
{
    private readonly ILogger<ProductListViewModel> _logger;

    public ProductListViewModel(ILogger<ProductListViewModel> logger, ...)
    {
        _logger = logger;
    }

    private async Task LoadProductsAsync()
    {
        try { /* ... */ }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to load products");
            // show error to user via IDialogService
        }
    }
}
```
