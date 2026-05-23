# Data Binding Deep Reference

## Binding Modes

| Mode | Direction | Typical Use |
|---|---|---|
| `OneWay` | Source → Target | Display-only labels, read-only fields |
| `TwoWay` | Source ↔ Target | TextBox, editable forms |
| `OneWayToSource` | Target → Source | Rarely used; write-only scenarios |
| `OneTime` | Source → Target (once) | Static data, loaded once |
| `Default` | Depends on property | Each DependencyProperty defines its default |

TextBox.Text defaults to TwoWay. TextBlock.Text defaults to OneWay. Know this — it's a common
interview question and a source of real confusion.

---

## UpdateSourceTrigger

Controls *when* TwoWay binding writes back to the source.

| Value | Behavior |
|---|---|
| `PropertyChanged` | On every keystroke |
| `LostFocus` | When user tabs/clicks away (default for TextBox) |
| `Explicit` | Only when `BindingExpression.UpdateSource()` is called |

```xml
<!-- Validate on every keystroke -->
<TextBox Text="{Binding SearchTerm, UpdateSourceTrigger=PropertyChanged}" />
```

**Production tip:** Use `PropertyChanged` for search/filter fields. Use `LostFocus` for
forms where live validation is annoying. Use `Explicit` for wizards where you validate
on "Next" button click.

---

## MultiBinding

Combine multiple sources into one target value.

```xml
<TextBlock>
    <TextBlock.Text>
        <MultiBinding StringFormat="{}{0} {1}">
            <Binding Path="FirstName" />
            <Binding Path="LastName" />
        </MultiBinding>
    </TextBlock.Text>
</TextBlock>
```

For complex logic, implement `IMultiValueConverter`.

---

## Binding Internals (How It Actually Works)

When you write `{Binding Path=Name}`:

1. WPF creates a `BindingExpression` object.
2. It locates the `DataContext` by walking up the visual tree.
3. It uses reflection (or DependencyProperty if available) to read the property.
4. It subscribes to `INotifyPropertyChanged.PropertyChanged` to detect future changes.
5. When `PropertyChanged` fires with the property name, WPF re-reads and updates the target.

**Why does binding fail silently?** Because WPF treats binding errors as non-fatal. It logs
them to the Output window at Debug level. Always have the Output window open during development
and set `PresentationTraceSources.TraceLevel` to `High` for hard-to-find bugs.

```csharp
// In constructor or App.xaml.cs — enables verbose binding trace
PresentationTraceSources.DataBindingSource.Switch.Level = SourceLevels.Critical;
```

---

## Binding to Non-ViewModel Sources

### ElementName Binding
```xml
<Slider x:Name="OpacitySlider" Minimum="0" Maximum="1" Value="0.5" />
<TextBlock Opacity="{Binding ElementName=OpacitySlider, Path=Value}" Text="Hello" />
```

### RelativeSource Binding
```xml
<!-- Bind to a property of the same element -->
<Border Tag="{Binding RelativeSource={RelativeSource Self}, Path=ActualWidth}" />

<!-- Bind to an ancestor in the visual tree -->
<DataTemplate>
    <Button Command="{Binding DataContext.DeleteCommand,
        RelativeSource={RelativeSource AncestorType=ItemsControl}}"
        CommandParameter="{Binding}" />
</DataTemplate>
```

`RelativeSource AncestorType` is critical in DataTemplates where DataContext is the item,
but you need to reach the parent ViewModel's commands.

---

## StaticResource vs DynamicResource

| | StaticResource | DynamicResource |
|---|---|---|
| Resolved | At load time | At runtime |
| Performance | Faster | Slightly slower |
| Theme switching | No | Yes |
| Use for | Fixed styles/colors | Themeable, changeable resources |

---

## Common Binding Pitfalls

1. **Binding to a non-public property** — WPF can't bind to private/internal properties via reflection in some trust contexts.
2. **Forgetting INotifyPropertyChanged** — property changes silently don't update UI.
3. **Binding in a DataTemplate without RelativeSource** — DataContext is the item, not the parent ViewModel.
4. **Modifying ObservableCollection on a background thread** — throws `InvalidOperationException`. Always dispatch.
5. **Binding Path typos** — fail silently. Check Output window.
