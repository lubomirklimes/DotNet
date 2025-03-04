# ConfigureAwait a synchronizační kontext

`ConfigureAwait(false)` je jedno z nejčastěji diskutovaných témat v .NET asynchronním programování. Nesprávné pochopení vede buď k deadlockům v UI aplikacích, nebo k zbytečnému kódu v serverových aplikacích, kde `ConfigureAwait(false)` nemá žádný efekt.

## 1. Koncept

Když napíšete `await someTask`, .NET po dokončení úlohy potřebuje vědět, **na jakém vlákně nebo kontextu pokračovat**. K tomu slouží `SynchronizationContext`.

- **WPF / WinForms / ASP.NET classic**: existuje `SynchronizationContext`, který zajišťuje, že pokračování běží na UI vlákně nebo na původním HTTP vlákně.
- **ASP.NET Core / konzolová aplikace**: žádný `SynchronizationContext` neexistuje; pokračování probíhá na libovolném vlákně z ThreadPoolu.

`ConfigureAwait(false)` říká: "nechci se vracet na původní kontext – pokračuj na ThreadPoolu". Eliminuje jeden marshalling a může zlepšit výkon v tight smyčkách nebo knihovních kódech.

## 2. Příklad

### Deadlock v UI aplikaci

```csharp
// ❌ Klasický deadlock ve WPF / WinForms / ASP.NET Classic
private void Button_Click(object sender, EventArgs e)
{
    // Synchronní čekání na async metodu v UI vlákně
    var data = StahniAsync().Result; // DEADLOCK
    // Co se děje:
    // 1) UI vlákno čeká na dokončení tasku
    // 2) Task čeká na pokračování na UI vlákně (SynchronizationContext)
    // 3) UI vlákno je zablokované → pokračování se nikdy nespustí → deadlock
}

public async Task<string> StahniAsync()
{
    return await _client.GetStringAsync("https://example.com");
    // Pokračování chce běžet na UI vlákně (kde byl zavolán await)
    // Ale UI vlákno čeká na .Result → deadlock
}
```

```csharp
// ✅ Řešení: použijte ConfigureAwait(false) v knihovní metodě
public async Task<string> StahniAsync()
{
    return await _client.GetStringAsync("https://example.com")
        .ConfigureAwait(false); // Pokračování na ThreadPoolu, ne na UI vlákně
    // Teď .Result() neblokuje UI vlákno, protože pokračování nečeká na UI
}
```

### Správné použití v knihovním kódu

```csharp
// Knihovna (NuGet balíček, sdílená utility třída)
// Pravidlo: ConfigureAwait(false) na každý await – knihovna nezná kontext volajícího
public class HttpDataClient(HttpClient http)
{
    public async Task<T?> ZiskejAsync<T>(string url, CancellationToken ct = default)
    {
        using var response = await http.GetAsync(url, ct).ConfigureAwait(false);
        response.EnsureSuccessStatusCode();

        var json = await response.Content.ReadAsStringAsync(ct).ConfigureAwait(false);
        return JsonSerializer.Deserialize<T>(json);
    }
}
```

### ASP.NET Core – ConfigureAwait nemá efekt

```csharp
// ASP.NET Core nemá SynchronizationContext
// ConfigureAwait(false) je zde no-op, ale neškodí
[ApiController]
public class ProduktController(IProduktRepo repo) : ControllerBase
{
    [HttpGet("{id}")]
    public async Task<IActionResult> Get(int id, CancellationToken ct)
    {
        // ConfigureAwait(false) zde nic nemění – ale je dobrá praxe pro konzistenci
        var produkt = await repo.NactiAsync(id, ct).ConfigureAwait(false);
        return produkt is null ? NotFound() : Ok(produkt);
    }
}
```

## 3. Kdy použít

- **Knihovní kód (NuGet, shared utilities)**: vždy `ConfigureAwait(false)` – knihovna neví, v jakém kontextu bude volána.
- **ASP.NET Core aplikace**: `ConfigureAwait(false)` je optional (context neexistuje), ale konzistentní použití je dobrá praxe.
- **WPF / WinForms – vrstva UI (code-behind, ViewModel)**: `await` bez `ConfigureAwait(false)` – pokračování musí být na UI vlákně pro aktualizaci UI.
- **WPF / WinForms – servisní vrstva**: `ConfigureAwait(false)` pro prevenci deadlocku, pokud někdo volá z UI synchronně.

## 4. Časté chyby

- ❌ **`task.Result` nebo `task.Wait()` ve WPF/WinForms** – deadlock jak je popsáno výše; nikdy neblokujte synchronně async kód s UI kontextem. → Viz [Async_Sync_Interop.md](Async_Sync_Interop.md) pro podrobnou anatomii deadlocku a bezpečné vzory přechodu sync↔async.
- ❌ **ConfigureAwait(false) v code-behind WPF po přístupu k UI** – pokud po `ConfigureAwait(false)` přistupujete k UI prvkům, dostanete `InvalidOperationException` (přístup z jiného vlákna).
- ❌ **Vynechání ConfigureAwait v jednom awaitu ve smyčce** – stačí jeden `await` bez `ConfigureAwait(false)` a kontext se zachytí zpět.

---

<details>
<summary>Deep dive: jak SynchronizationContext funguje interně</summary>

### SynchronizationContext a jeho implementace

`SynchronizationContext.Current` je thread-local hodnota. Každý `await` (výchozí `TaskAwaiter`) při zahájení čekání zavolá `SynchronizationContext.Current.Post(continuation)` pro naplánování pokračování.

- **UI kontext (WPF)**: `DispatcherSynchronizationContext` – `Post` zařadí pokračování do Dispatcher fronty (= UI thread message loop).
- **ASP.NET classic**: `AspNetSynchronizationContext` – zajišťoval přístup k `HttpContext` z pokračování.
- **ASP.NET Core**: žádný kontext – `SynchronizationContext.Current` je `null`; `await` pokračuje na ThreadPoolu.

### ConfigureAwait(false) vs ValueTask

`ConfigureAwait(false)` vrátí `ConfiguredTaskAwaitable`, který při `GetAwaiter()` nastaví `continueOnCapturedContext = false`. Výsledné pokračování se naplánuje přímo na `ThreadPool`, ne přes `SynchronizationContext.Post`.

### Analyzátory kódu

Přidejte Roslyn analyzátor pro automatické hlídání chybějícího `ConfigureAwait`:

```xml
<PackageReference Include="ConfigureAwaitChecker.Analyzer" Version="5.*" PrivateAssets="all" />
```

Nebo nastavte EditorConfig pravidlo: `dotnet_diagnostic.CA2007.severity = warning`.

</details>

**Další krok:** [Orchestrace úloh: Task.WhenAll, Task.WhenAny a timeouty](Task_Orchestration.md)
