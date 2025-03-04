# WPF (Windows Presentation Foundation)

WPF umožňuje tvorbu moderních desktopových aplikací se škálovatelným UI založeným na **XAML** a vzoru **MVVM**.

## Vytvoření projektu

```
dotnet new wpf -n MojeWpfApp
```

## XAML – popis rozhraní

```xml
<Window x:Class="MojeApp.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        Title="Moje aplikace" Height="300" Width="400">
    <StackPanel Margin="10">
        <TextBox x:Name="txtJmeno" Margin="0,0,0,8"/>
        <Button Content="Pozdrav" Click="BtnPozdrav_Click"/>
        <TextBlock x:Name="lblVystup" Margin="0,8,0,0"/>
    </StackPanel>
</Window>
```

```csharp
// MainWindow.xaml.cs
private void BtnPozdrav_Click(object sender, RoutedEventArgs e)
{
    lblVystup.Text = $"Ahoj, {txtJmeno.Text}!";
}
```

## Data Binding

Binding propojuje UI prvky s datovým modelem bez réčního kódu:

```xml
<TextBlock Text="{Binding Jmeno}"/>
<TextBox Text="{Binding Email, UpdateSourceTrigger=PropertyChanged}"/>
```

```csharp
public class OsobaViewModel : INotifyPropertyChanged
{
    private string _jmeno;
    public string Jmeno
    {
        get => _jmeno;
        set { _jmeno = value; OnPropertyChanged(); }
    }

    public event PropertyChangedEventHandler? PropertyChanged;
    protected void OnPropertyChanged([CallerMemberName] string? name = null)
        => PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(name));
}
```

```csharp
// V okně nastav DataContext
DataContext = new OsobaViewModel { Jmeno = "Petr" };
```

## MVVM vzor

| Vrstva | Popis |
|---|---|
| **Model** | Data a business logika |
| **ViewModel** | Připraví data pro View, zpracovává příkazy |
| **View** | XAML – pouze zobrazuje, neobsahuje logiku |

ViewModel komunikuje s View přes `Binding` a `INotifyPropertyChanged` – nikdy nezísáví na View přímo.

---

<details>
<summary>Volitelné: Commands, Styles a Triggers</summary>

**ICommand pro tlačítka bez code-behind:**

```csharp
public ICommand PozdravCommand { get; } =
    new RelayCommand(() => MessageBox.Show("Ahoj!"));
```

```xml
<Button Content="Pozdrav" Command="{Binding PozdravCommand}"/>
```

**Styly a triggery:**

```xml
<Button Content="OK">
    <Button.Style>
        <Style TargetType="Button">
            <Setter Property="Background" Value="LightBlue"/>
            <Style.Triggers>
                <Trigger Property="IsMouseOver" Value="True">
                    <Setter Property="Background" Value="SteelBlue"/>
                </Trigger>
            </Style.Triggers>
        </Style>
    </Button.Style>
</Button>
```

</details>

---

**Další krok:** [MAUI – multiplatformní desktopové aplikace](MAUI.md)
