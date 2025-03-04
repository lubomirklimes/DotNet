# .NET MAUI – multiplatformní aplikace

.NET MAUI (Multi-platform App UI) umožňuje psaní jedné kódbaze pro **Windows, macOS, Android a iOS**.

## Vytvoření projektu

```
dotnet new maui -n MojeMauiApp
```

## Struktura projektu

```
MojeMauiApp/
  MauiProgram.cs         - konfigurace aplikace
  App.xaml / App.xaml.cs - koren aplikace
  MainPage.xaml          - hlavní stránka
  Resources/             - obrázky, fonty, styly
```

## UI v XAML

```xml
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             Title="Hlavní stránka">
    <VerticalStackLayout Padding="16" Spacing="8">
        <Label Text="Zadej jméno:"/>
        <Entry x:Name="EntryJmeno" Placeholder="Jméno"/>
        <Button Text="Pozdrav" Clicked="OnPozdravClicked"/>
        <Label x:Name="LabelVystup"/>
    </VerticalStackLayout>
</ContentPage>
```

```csharp
private void OnPozdravClicked(object sender, EventArgs e)
{
    LabelVystup.Text = $"Ahoj, {EntryJmeno.Text}!";
}
```

## Navigace mezi stránkami

```csharp
// Registrace v MauiProgram.cs
builder.Services.AddTransient<DetailPage>();
builder.Services.AddTransient<DetailViewModel>();

// Navigace
await Shell.Current.GoToAsync($"//detail?id={produkt.Id}");
```

## MVVM s CommunityToolkit.Mvvm

```csharp
// NuGet: CommunityToolkit.Mvvm
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

partial class MainViewModel : ObservableObject
{
    [ObservableProperty]
    private string jmeno = "";

    [RelayCommand]
    private void Pozdrav()
        => VysledekText = $"Ahoj, {Jmeno}!";

    [ObservableProperty]
    private string vysledekText = "";
}
```

`[ObservableProperty]` a `[RelayCommand]` generují potřebný kód automaticky.

---

<details>
<summary>Volitelné: Platform-specifický kód a nativní funkce</summary>

Pro přístup k platform-specifickým API (GPS, kamera, ...) použij:

```csharp
// Maui.Essentials (vestavěný)
var location = await Geolocation.GetLastKnownLocationAsync();
await Launcher.OpenAsync("https://example.com");

// Platform-specifický kód
#if ANDROID
    // Android-specific kód
#elif IOS
    // iOS-specific kód
#endif
```

</details>

---

**Další krok:** [Migrace z Xamarin na .NET MAUI](Xamarin_migrace.md)
