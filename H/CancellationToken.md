# CancellationToken – zrušení asynchronních operací

`CancellationToken` je mechanismus, který umožňuje volajícímu signalizovat asynchronní operaci: "přestaň pracovat, výsledek už nepotřebuji". Bez podpory cancellation nelze efektivně zvládat timeouty, odpojené klienty ani graceful shutdown.

## 1. Koncept

`CancellationToken` je hodnotový typ (struct), který nese referenci na `CancellationTokenSource`. Volající vytvoří `CancellationTokenSource`, předá `Token` do asynchronní metody a může kdykoli zavolat `Cancel()`. Metoda pravidelně kontroluje stav tokenu (nebo používá metody, které to dělají automaticky) a vyhodí `OperationCanceledException`.

Klíčová vlastnost: cancellation je **kooperativní**. Metoda se musí vzdát dobrovolně – token jen signalizuje záměr zrušit, nevyjme vlákno násilím.

## 2. Příklad

### Základní použití

```csharp
// Volající vytvoří zdroj tokenu
var cts = new CancellationTokenSource();

// Předá token do metody
var task = StahniDataAsync("https://example.com", cts.Token);

// Kdykoli může zrušit
cts.Cancel(); // Nastaví IsCancellationRequested = true

try
{
    await task;
}
catch (OperationCanceledException)
{
    Console.WriteLine("Operace byla zrušena.");
}

// Implementace metody – token propagujte do všech vnitřních volání
public async Task<string> StahniDataAsync(string url, CancellationToken ct = default)
{
    using var client = new HttpClient();
    // GetStringAsync přijme token a vyhodí OperationCanceledException při zrušení
    return await client.GetStringAsync(url, ct);
}
```

### Timeout

```csharp
// CancelAfter nastaví automatické zrušení po uplynutí doby
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(10));

try
{
    var data = await StahniDataAsync(url, cts.Token);
}
catch (OperationCanceledException) when (cts.IsCancellationRequested)
{
    Console.WriteLine("Timeout po 10 sekundách.");
}
```

### Propojení tokenů (LinkedTokenSource)

```csharp
// Scénář: request má vlastní timeout + aplikace se vypíná (stoppingToken)
public async Task ZpracujRequestAsync(HttpContext context, CancellationToken stoppingToken)
{
    // Zrušíme pokud uplyne timeout NEBO aplikace se vypíná – co nastane dříve
    using var requestCts = CancellationTokenSource.CreateLinkedTokenSource(
        context.RequestAborted,  // token klienta (odpojení prohlížeče)
        stoppingToken);           // token aplikace (SIGTERM, restart)
    requestCts.CancelAfter(TimeSpan.FromSeconds(30)); // navíc timeout 30s

    await ByznysovaLogika(requestCts.Token);
}
```

### Ruční kontrola v CPU-bound kódu

```csharp
// Pro synchronní nebo CPU-bound práci kontrolujte token ručně
public async Task<List<Vysledek>> ZpracujDavskyAsync(
    IEnumerable<Polozka> polozky, CancellationToken ct)
{
    var vysledky = new List<Vysledek>();

    foreach (var polozka in polozky)
    {
        // ThrowIfCancellationRequested vyhodí OperationCanceledException pokud je zrušeno
        ct.ThrowIfCancellationRequested();

        var vysledek = await ZpracujPolozkuAsync(polozka, ct);
        vysledky.Add(vysledek);
    }

    return vysledky;
}
```

### Graceful shutdown v ASP.NET Core

```csharp
// IHostedService dostane stoppingToken z frameworku při SIGTERM nebo Ctrl+C
public class ZpracovatelFronty(IFrontaService fronta, ILogger<ZpracovatelFronty> log)
    : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var zprava in fronta.CtiAsync(stoppingToken))
        {
            try
            {
                await ZpracujAsync(zprava, stoppingToken);
            }
            catch (OperationCanceledException)
            {
                log.LogInformation("Zpracování zastaveno.");
                return;
            }
        }
    }
}
```

## 3. Kdy použít

- **Všechny async veřejné metody** – vždy přidejte `CancellationToken ct = default` jako poslední parametr. Výchozí hodnota `default` je `CancellationToken.None` (nikdy nezrušeno).
- **HTTP requesty** – předejte `HttpContext.RequestAborted` – automaticky se zruší při odpojení klienta.
- **Background services** – používejte `stoppingToken` z `BackgroundService.ExecuteAsync`.
- **Timeouty** – `CancellationTokenSource(TimeSpan)` je čistší než `Task.WhenAny` s `Task.Delay`.
- **Uživatelsky inicializované zrušení** – tlačítko "Zrušit" v UI, stop operace, atd.

## 4. Časté chyby

- ❌ **Ignorování tokenu** – přijmout `CancellationToken` jako parametr a nikam ho nepředávat. Metoda pak nelze zrušit přes token.
- ❌ **Zachycení `OperationCanceledException` bez re-throw** – spolknutí výjimky způsobí, že volající neví o zrušení; vždy re-throw nebo alespoň logujte.
- ❌ **`CancellationTokenSource` bez `using`** – drží OS handle; vždy `using` nebo `Dispose()`.
- ❌ **Kontrola `IsCancellationRequested` místo `ThrowIfCancellationRequested`** – ruční `if` je verbose; `ThrowIfCancellationRequested()` je idiomatické a kratší.
- ❌ **Předání `CancellationToken.None` explicitně** – signalizuje záměrné ignorování cancellation; pokud token máte, předejte ho. Místo `CancellationToken.None` použijte `default`.

---

<details>
<summary>Deep dive: interní implementace a výkonové aspekty</summary>

### Jak token signalizuje zrušení

`CancellationTokenSource` obsahuje `volatile int _state`. `Cancel()` atomicky změní stav a spustí všechny registrované callbacks přes `CancellationToken.Register()`. Každý `await` na cancellable operaci interně registruje callback, který zruší awaitable.

### Registrace callbacku

```csharp
// Spuštění cleanup logiky při zrušení
using var registration = ct.Register(() =>
{
    log.LogWarning("Token zrušen – cleanup...");
    _nativniZdroj.Zrus();
});
// Registration se automaticky odregistruje při Dispose
```

### Výkon – proč ct = default místo null

`CancellationToken` je struct; `default` je `CancellationToken.None` (nezrušitelný token s `CanBeCanceled = false`). Kontroly `ct.ThrowIfCancellationRequested()` jsou pak no-op bez alokace – ideální pro hot paths.

### Linked tokens a memory

`CreateLinkedTokenSource` alokuje nový `CancellationTokenSource` s interními registracemi. Vždy volejte `Dispose()` přes `using` – jinak dojde k memory leaku přes registrace na parent tokenech.

</details>

**Další krok:** [ConfigureAwait a synchronizační kontext](ConfigureAwait.md)
