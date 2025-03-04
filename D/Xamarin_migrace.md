# Xamarin a .NET MAUI – migrace a přechod

Xamarin byl první cross-platform mobilní framework Microsoftu pro C#. Od roku 2024 je Xamarin officially End-of-Life a nahrazuje ho **.NET MAUI** (Multi-platform App UI), které přináší modernizované API, lepší výkon a podporu nejen pro mobilní, ale i desktopové platformy.

## 1. Koncept

### Proč Microsoft přešel od Xamarin k MAUI

Xamarin.Forms byl postaven jako thin wrapper nad nativními UI komponentami. Každý renderer byl zvlášť pro Android, iOS a UWP, a sdílet kód mezi platformami bylo složité. MAUI toto řeší:

- **Jeden projekt** místo oddělených Android/iOS projektů.
- **Handlers místo rendererů** – lehčí, snáze přizpůsobitelné.
- **Hot Reload** – změny XAML se projeví bez restart.
- **Lepší startup výkon** díky modernizaci runtime bootstrappingu.

### Stav platforem

| Framework | Stav (2024) | Platformy |
|-----------|-------------|-----------|
| Xamarin.Android | EOL | Android |
| Xamarin.iOS | EOL | iOS |
| Xamarin.Forms | EOL | Android, iOS, UWP |
| **.NET MAUI** | Aktivní vývoj | Android, iOS, Windows, macOS |

> Pro nové projekty vždy volte .NET MAUI.

## 2. Příklad

### Nový MAUI projekt

```bash
dotnet new maui -n MojeMauiApp
cd MojeMauiApp
dotnet run --framework net9.0-android
```

### Klíčové mapování Xamarin → MAUI

```csharp
// Xamarin.Forms – registrace aplikace
public class App : Xamarin.Forms.Application
{
    public App() { MainPage = new NavigationPage(new MainPage()); }
}

// MAUI – registrace aplikace v MauiProgram.cs
var builder = MauiApp.CreateBuilder();
builder
    .UseMauiApp<App>()
    .ConfigureFonts(fonts =>
    {
        fonts.AddFont("OpenSans-Regular.ttf", "OpenSansRegular");
    });
```

```xml
<!-- Xamarin.Forms XAML namespace -->
xmlns="http://xamarin.com/schemas/2014/forms"

<!-- MAUI XAML namespace -->
xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
```

### Migrace vlastního rendereru na Handler

```csharp
// Xamarin.Forms – vlastní renderer pro zaoblená tlačítka
public class ZaobleneButtonRenderer : ButtonRenderer
{
    protected override void OnElementChanged(ElementChangedEventArgs<Button> e)
    {
        base.OnElementChanged(e);
        if (Control != null) Control.Layer.CornerRadius = 12;
    }
}

// MAUI – Handler (čistší přístup, méně boilerplate)
public class ZaobleneButtonHandler : ButtonHandler
{
    protected override void ConnectHandler(MauiButton platformView)
    {
        base.ConnectHandler(platformView);
        platformView.Layer.CornerRadius = 12; // iOS specifické
    }
}

// Registrace v MauiProgram.cs
builder.ConfigureMauiHandlers(handlers =>
{
    handlers.AddHandler<Button, ZaobleneButtonHandler>();
});
```

### Platformě-specifický kód

```csharp
// MAUI – podmíněný kód bez #if direktiv
#if ANDROID
    // Jen pro Android
    var result = await Permissions.RequestAsync<Permissions.LocationWhenInUse>();
#elif IOS
    // Jen pro iOS
    AVAudioSession.SharedInstance().RequestRecordPermission(...);
#endif

// Nebo přes abstrakci:
public interface IToastService { void Show(string zprava); }

// Android implementace (Platforms/Android/ToastService.cs)
public class ToastService : IToastService
{
    public void Show(string zprava) =>
        Toast.MakeText(Android.App.Application.Context, zprava, ToastLength.Short)?.Show();
}
```

## 3. Kdy použít

- **Nová aplikace** → vždy .NET MAUI; Xamarin nelze doporučit pro nové projekty.
- **Existující Xamarin.Forms** → migrujte postupně: nejdříve aktualizujte namespace, pak přepište renderery na handlery.
- **Existující Xamarin Native** (Android/iOS bez Forms) → zvažte přechod, ale migrace je pracnější; nativní API zůstávají podobná.

## 4. Časté chyby při migraci

- ❌ **Přímé kopírování XAML bez změny namespace** – Xamarin a MAUI mají odlišné namespace URI; XAML bez úpravy zkompiluje, ale chování se liší.
- ❌ **Neaktualizovaná NuGet balíčky** – Xamarin knihovny (Xamarin.Essentials) jsou nahrazeny MAUI ekvivalenty (.NET MAUI Essentials je součástí frameworku); neinstalujte Xamarin NuGet do MAUI projektu.
- ❌ **Záměna `Device.BeginInvokeOnMainThread` s `MainThread.InvokeOnMainThreadAsync`** – v MAUI preferujte async variantu.
- ❌ **Přepsání celé aplikace najednou** – migrujte stránku po stránce; udržujte fungující stav po každém kroku.

---

<details>
<summary>Deep dive: Shell navigace a MVVM s Community Toolkit</summary>

### Shell navigace

MAUI Shell poskytuje URL-based navigaci podobnou webovým aplikacím:

```csharp
// Registrace tras v AppShell.xaml.cs
Routing.RegisterRoute("detail", typeof(DetailPage));
Routing.RegisterRoute("nastaveni", typeof(NastaveniPage));

// Navigace s parametry
await Shell.Current.GoToAsync($"detail?id={produkt.Id}&zdroj=seznam");

// Přijetí parametrů v cílové stránce
[QueryProperty(nameof(ProduktId), "id")]
public partial class DetailPage : ContentPage
{
    public string ProduktId { get; set; } = string.Empty;
}
```

### MVVM s CommunityToolkit.Mvvm

```csharp
// NuGet: CommunityToolkit.Mvvm
// Source generátory eliminují boilerplate INotifyPropertyChanged

[ObservableObject]
public partial class DetailViewModel
{
    [ObservableProperty]
    private string _nazev = string.Empty; // Generuje property Nazev + OnNazevChanged

    [ObservableProperty]
    private bool _nacitani;

    [RelayCommand]
    private async Task NactiAsync(CancellationToken ct)
    {
        Nacitani = true;
        try { Nazev = await _repo.ZiskejNazevAsync(ProduktId, ct); }
        finally { Nacitani = false; }
    }
}
```

Dokumentace migrace: [learn.microsoft.com/dotnet/maui/migration](https://learn.microsoft.com/dotnet/maui/migration)

</details>

**Další krok:** [Práce s GPS, senzory a notifikacemi](Mobile_Features.md)
