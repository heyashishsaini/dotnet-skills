# WPF Performance & Memory Reference

## UI Virtualization

WPF renders only visible items when virtualization is enabled. For 10,000-row lists,
this is the difference between instant and unusable.

```xml
<!-- ListBox — virtualization is ON by default -->
<ListBox ItemsSource="{Binding Items}" />

<!-- DataGrid — also on by default, but watch for GroupBy disabling it -->
<DataGrid ItemsSource="{Binding Items}" EnableRowVirtualization="True" />

<!-- If you wrap in a ScrollViewer manually, virtualization breaks. Don't. -->
<!-- WRONG: -->
<ScrollViewer>
    <ItemsControl ItemsSource="{Binding Items}" />   <!-- no virtualization -->
</ScrollViewer>

<!-- RIGHT: Use VirtualizingStackPanel explicitly -->
<ItemsControl ItemsSource="{Binding Items}">
    <ItemsControl.ItemsPanel>
        <ItemsPanelTemplate>
            <VirtualizingStackPanel />
        </ItemsPanelTemplate>
    </ItemsControl.ItemsPanel>
</ItemsControl>
```

---

## Freezables

`Freezable` objects (Brush, Geometry, Animation) can be frozen to become immutable and
shared across threads without marshaling overhead.

```csharp
var brush = new SolidColorBrush(Colors.Red);
brush.Freeze(); // Now immutable — can be shared, no thread affinity
```

**When to freeze:** Any `Freezable` you create once and never modify. Frozen objects
are also excluded from the change notification system, which reduces overhead.

---

## Rendering Pipeline Awareness

WPF uses a **retained mode** rendering system. The visual tree is submitted to the
`MilCore` compositor via `DrawingContext`. Key implications:

- `Measure` + `Arrange` passes happen on the UI thread.
- Heavy custom `OnRender` code causes frame drops.
- `BitmapCache` can cache a rendered Visual as a bitmap — useful for complex but static visuals.

```xml
<Border>
    <Border.CacheMode>
        <BitmapCache />
    </Border.CacheMode>
    <!-- Complex content that doesn't change often -->
</Border>
```

---

## Common Memory Leak Patterns

### 1. Event Handler Leak (most common)

```csharp
// LEAK: static event holds reference to ViewModel
public class MyViewModel
{
    public MyViewModel()
    {
        SomeStaticClass.SomeStaticEvent += OnEvent; // ViewModel can never be GC'd
    }
    // No Unsubscribe!
}

// FIX: Unsubscribe in a cleanup method, or use WeakEventManager
WeakEventManager<SomeStaticClass, EventArgs>.AddHandler(
    null, nameof(SomeStaticClass.SomeStaticEvent), OnEvent);
```

### 2. CollectionChanged Leak

```csharp
// LEAK: subscribing to CollectionChanged without unsubscribing
public MyViewModel(ObservableCollection<Item> sharedList)
{
    sharedList.CollectionChanged += OnItemsChanged; // leak if sharedList outlives this VM
}
```

### 3. DispatcherTimer Not Stopped

```csharp
// LEAK: timer holds reference, ViewModel never GC'd
_timer = new DispatcherTimer();
_timer.Tick += OnTick;
_timer.Start();
// FIX: _timer.Stop() when ViewModel is done
```

### 4. DataContext Not Cleared

If a View's DataContext holds a ViewModel with long-lived subscriptions, and you close the
View but don't clear DataContext, the ViewModel stays alive.

```csharp
// In Window or UserControl code-behind:
protected override void OnClosed(EventArgs e)
{
    base.OnClosed(e);
    if (DataContext is IDisposable vm) vm.Dispose();
    DataContext = null;
}
```

---

## Profiling Tools

| Tool | Use For |
|---|---|
| Visual Studio Diagnostic Tools | CPU/memory snapshot during debug |
| dotMemory (JetBrains) | Heap analysis, leak detection |
| PerfView | GC pressure, ETW traces |
| WPF Performance Suite (Perforator, Visual Profiler) | Rendering layer analysis |
| Snoop / WPF Inspector | Live visual tree and binding inspection |

**Snoop** is the most useful day-to-day tool. It lets you inspect the live visual tree,
check DataContext values, and spot binding errors at runtime. Install it.
