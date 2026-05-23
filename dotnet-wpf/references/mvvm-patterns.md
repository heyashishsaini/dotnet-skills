# MVVM Patterns Reference

## ViewModelBase

Every ViewModel inherits from a shared base that implements `INotifyPropertyChanged`.

```csharp
public abstract class ViewModelBase : INotifyPropertyChanged
{
    public event PropertyChangedEventHandler? PropertyChanged;

    protected void OnPropertyChanged([CallerMemberName] string? name = null)
        => PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(name));

    protected bool SetProperty<T>(ref T field, T value, [CallerMemberName] string? name = null)
    {
        if (EqualityComparer<T>.Default.Equals(field, value)) return false;
        field = value;
        OnPropertyChanged(name);
        return true;
    }
}
```

**Why SetProperty?** Avoids raising PropertyChanged when value hasn't changed — prevents
unnecessary UI re-renders and infinite binding loops.

---

## RelayCommand

The standard ICommand implementation. No external library needed.

```csharp
public class RelayCommand : ICommand
{
    private readonly Action<object?> _execute;
    private readonly Func<object?, bool>? _canExecute;

    public RelayCommand(Action<object?> execute, Func<object?, bool>? canExecute = null)
    {
        _execute = execute ?? throw new ArgumentNullException(nameof(execute));
        _canExecute = canExecute;
    }

    public event EventHandler? CanExecuteChanged
    {
        add => CommandManager.RequerySuggested += value;
        remove => CommandManager.RequerySuggested -= value;
    }

    public bool CanExecute(object? parameter) => _canExecute?.Invoke(parameter) ?? true;
    public void Execute(object? parameter) => _execute(parameter);
}
```

**Production note:** `CommandManager.RequerySuggested` re-evaluates `CanExecute` on any UI
interaction. For expensive `CanExecute` logic, use `RaiseCanExecuteChanged()` manually instead.

---

## RelayCommand<T> (Generic)

```csharp
public class RelayCommand<T> : ICommand
{
    private readonly Action<T?> _execute;
    private readonly Func<T?, bool>? _canExecute;

    public RelayCommand(Action<T?> execute, Func<T?, bool>? canExecute = null)
    {
        _execute = execute;
        _canExecute = canExecute;
    }

    public event EventHandler? CanExecuteChanged
    {
        add => CommandManager.RequerySuggested += value;
        remove => CommandManager.RequerySuggested -= value;
    }

    public bool CanExecute(object? parameter) => _canExecute?.Invoke((T?)parameter) ?? true;
    public void Execute(object? parameter) => _execute((T?)parameter);
}
```

---

## Navigation Patterns

### Option 1: Simple Frame Navigation (small apps)

```xml
<!-- MainWindow.xaml -->
<Frame x:Name="MainFrame" NavigationUIVisibility="Hidden" />
```

```csharp
// NavigationService wrapper
public class NavigationService
{
    private Frame _frame;
    public NavigationService(Frame frame) => _frame = frame;
    public void NavigateTo<T>() where T : Page, new() => _frame.Navigate(new T());
}
```

**Problem:** Pages are tightly coupled to `System.Windows.Controls.Page`. ViewModels
end up aware of page types. Acceptable for small apps only.

---

### Option 2: ViewModel-First Navigation (production)

The ViewModel controls navigation state. The View reacts via DataTemplates.

```xml
<!-- MainWindow.xaml -->
<ContentControl Content="{Binding CurrentViewModel}">
    <ContentControl.Resources>
        <DataTemplate DataType="{x:Type vm:DashboardViewModel}">
            <views:DashboardView />
        </DataTemplate>
        <DataTemplate DataType="{x:Type vm:SettingsViewModel}">
            <views:SettingsView />
        </DataTemplate>
    </ContentControl.Resources>
</ContentControl>
```

```csharp
// MainViewModel.cs
public class MainViewModel : ViewModelBase
{
    private ViewModelBase _currentViewModel;
    public ViewModelBase CurrentViewModel
    {
        get => _currentViewModel;
        set => SetProperty(ref _currentViewModel, value);
    }

    public MainViewModel()
    {
        CurrentViewModel = new DashboardViewModel();
    }

    public void NavigateTo(ViewModelBase vm) => CurrentViewModel = vm;
}
```

**Why this is production-grade:**
- ViewModel never references a View or Page type
- Navigation is testable (just check `CurrentViewModel` type)
- DataTemplate wires the View to ViewModel automatically — pure MVVM

---

## Messenger / Event Aggregator

For ViewModels that need to communicate without direct references.

```csharp
// Simple messenger (no library)
public static class Messenger
{
    private static readonly Dictionary<Type, List<Action<object>>> _handlers = new();

    public static void Subscribe<T>(Action<T> handler)
    {
        var type = typeof(T);
        if (!_handlers.ContainsKey(type)) _handlers[type] = new();
        _handlers[type].Add(msg => handler((T)msg));
    }

    public static void Send<T>(T message)
    {
        if (_handlers.TryGetValue(typeof(T), out var handlers))
            foreach (var h in handlers) h(message!);
    }
}
```

**Common use case:** After saving a record in `EditViewModel`, notify `ListViewModel`
to refresh — without them knowing about each other.

**Warning:** Static messengers can cause memory leaks if handlers aren't unsubscribed.
Use weak references or a proper library (CommunityToolkit.Mvvm, Prism) in production.
