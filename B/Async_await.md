# Async/await – základy

`async` a `await` jsou klíčová slova C# pro **asynchronní programování bez blokování vlákna**. Umožňují psát neblokující kód, který vypadá jako synchronní – bez callbacků a ručního řízení vláken.

## 1. Koncept

Synchronní volání blokuje vlákno po celou dobu operace (čekání na síť, disk, DB). Asynchronní volání vlákno **uvolní** při čekání – ThreadPool ho přidělí jinému requestu. Po dokončení operace se provádění vrátí tam, kde skončilo.

```
Synchronní:  [vlákno blokováno 200ms čekáním na DB]
Async:       [vlákno volné pro jiné requesty] → [DB vrátí] → [pokračování]
```

Pravidla:
- Metoda označená `async` smí obsahovat `await`
- `await` lze použít na jakýkoli `Task`, `Task<T>`, `ValueTask<T>` nebo `IAsyncEnumerable<T>`
- Návratový typ async metody je `Task`, `Task<T>`, nebo `void` (jen pro event handlery)

## 2. Příklad

### Základní async metoda

```csharp
// Synchronní – blokuje vlákno po dobu HTTP requestu
string StahniSync(string url)
{
    using var client = new HttpClient();
    return client.GetStringAsync(url).Result; // ❌ blokující
}

// ✅ Asynchronní – vlákno volné při čekání na odpověď
async Task<string> StahniAsync(string url, CancellationToken ct = default)
{
    using var client = new HttpClient();
    return await client.GetStringAsync(url, ct);
}
```

### Řetězení async volání

```csharp
// Každý await uvolní vlákno; celý řetěz je neblokující
async Task<UzivatelDetail> NactiDetailAsync(int id, CancellationToken ct)
{
    var uzivatel  = await _db.Uzivatele.FindAsync([id], ct);
    var objednavky = await _db.Objednavky
        .Where(o => o.UzivatelId == id)
        .ToListAsync(ct);

    return new UzivatelDetail(uzivatel!, objednavky);
}
```

### Paralelní čekání

```csharp
// Sekvenčně: 1s + 1s = 2s
var a = await ZiskejAsync("A"); // čeká 1s
var b = await ZiskejAsync("B"); // čeká další 1s

// ✅ Paralelně: celkový čas = nejpomalejší = 1s
var ukoly    = new[] { ZiskejAsync("A"), ZiskejAsync("B") };
var vysledky = await Task.WhenAll(ukoly);
string vysledekA = vysledky[0], vysledekB = vysledky[1];
// nebo přes Task.WhenAll přímo:
var (ta, tb) = (ZiskejAsync("A"), ZiskejAsync("B"));
await Task.WhenAll(ta, tb);
string ra = ta.Result, rb = tb.Result; // .Result je bezpečné PO await WhenAll
```

### Async v ASP.NET Core

```csharp
// Controller metoda – async po celou cestu od requestu po odpověď
[HttpGet("{id}")]
public async Task<IActionResult> GetUzivatel(int id, CancellationToken ct)
{
    var uzivatel = await _sluzba.NactiAsync(id, ct);
    return uzivatel is null ? NotFound() : Ok(uzivatel);
}
```

## 3. Kdy použít

- **Vždy pro I/O operace** – síťové requesty, čtení souborů, přístup k databázi.
- **ASP.NET Core controllery a Minimal API** – async po celé cestě zajistí maximální propustnost.
- **BackgroundService** – asynchronní smyčky pro zpracování front nebo pravidelných úloh.

**Nepoužívejte** `async` pro čistě CPU-bound operace (komprese, šifrování, matematika) – tam použijte `Task.Run` nebo `Parallel.ForEach`.

## 4. Časté chyby

- ❌ **`task.Result` nebo `task.Wait()`** – blokuje vlákno; v ASP.NET Core hrozí deadlock. Vždy `await task`.
- ❌ **`async void`** – výjimky z async void nejdou zachytit; používejte pouze pro event handlery.
- ❌ **Zbytečné `async`/`await` bez důvodu** – metoda, která jen vrací Task bez vlastního `await`, nepotřebuje být `async`; přidává zbytečnou state machine. Vraťte Task přímo.
- ❌ **Ignorování `CancellationToken`** – nepředávat token do downstream volání způsobí, že zrušení requestu kód nedohlídne a zbytečně pokračuje.

> Pro pokročilé vzory (ConfigureAwait, ValueTask, TaskCompletionSource, Channels) viz [H/Async_await_pokrocile.md](../H/Async_await_pokrocile.md).

**Další krok:** [CLR, sestavení a přehled runtime](CLR_overview.md)
