# Interakce mezi asynchronním a synchronním kódem

Smíchání async a synchronního kódu je zdroj jedněch z nejzáludnějších bugů v .NET: deadlocky, které se projeví jen v produkci, spolknuté výjimky a záhadné freeznutí UI. Základní pravidlo je jednoduché – neblokujte async kód synchronně – ale realita je složitější.

→ Viz [ConfigureAwait.md](ConfigureAwait.md) pro roli `SynchronizationContext` a `ConfigureAwait(false)` v prevenci deadlocků.

## 1. Koncept

### Proč blokování async kódu způsobuje deadlock

V prostředích se synchronizačním kontextem (WPF, WinForms, ASP.NET classic) platí:

1. `task.Result` nebo `task.Wait()` zablokuje aktuální vlákno a čeká na dokončení tasku.
2. Task po svém dokončení se pokusí pokračovat (continuation) na původním vlákně (synchronizační kontext).
3. Původní vlákno je zablokované v bodu 1 – čeká.
4. Continuation čeká na vlákno → vlákno čeká na continuation → **deadlock**.

V ASP.NET Core a konzolových aplikacích synchronizační kontext neexistuje, takže deadlock nenastane – continuation běží na ThreadPoolu. Kód však i tak zbytečně blokuje vlákno.

## 2. Příklad

### Nebezpečné vzory

```csharp
// ❌ Deadlock ve WPF / ASP.NET classic / WinForms
public string ZiskejDataSynchronne()
{
    return StahniAsync().Result;        // DEADLOCK
    return StahniAsync().GetAwaiter().GetResult(); // Stejný problém, jiná syntaxe
    Task.Run(StahniAsync).Result;       // Obejde kontext, ale blokuje vlákno
}

// ❌ task.Wait() – blokuje vlákno a v kontextovém prostředí způsobí deadlock
StahniAsync().Wait();
```

### Správné řešení – async celou cestou

```csharp
// ✅ Nejlepší: propagujte async celým call stackem
public async Task<string> ZiskejDataAsync(CancellationToken ct = default)
{
    return await StahniAsync(ct);
}

// V konzolové aplikaci nebo Main metodě:
static async Task Main() => await SpustAplikaciAsync();
```

### Kdy nelze jít async – bezpečné výjimky

```csharp
// Konstruktor nemůže být async – použijte factory metodu
public class Sluzba
{
    private Sluzba() { }

    public static async Task<Sluzba> VytvorAsync(CancellationToken ct = default)
    {
        var sluzba = new Sluzba();
        await sluzba.InitAsync(ct);
        return sluzba;
    }
}

// Implementace IDisposable.Dispose nemůže být async
// – zrušte operace přes CancellationToken, await v DisposeAsync
public class AsyncSluzba : IAsyncDisposable
{
    private readonly CancellationTokenSource _cts = new();

    public async ValueTask DisposeAsync()
    {
        _cts.Cancel();
        await _pozadiUloha; // Počkejte na korektní ukončení
        _cts.Dispose();
    }
}

// Použití:
await using var sluzba = new AsyncSluzba();
```

### Volání async z legacy synchronního kódu

Pokud **skutečně musíte** volat async kód synchronně (např. integrace se starými frameworky, které nepodporují async), použijte `Task.Run` pro izolaci od synchronizačního kontextu:

```csharp
// Poslední záchrana – pouze pokud není možné jít async celou cestou
public string ZiskejDataZLegacyKodu()
{
    // Task.Run spustí na ThreadPoolu BEZ synchronizačního kontextu → žádný deadlock
    return Task.Run(() => StahniAsync()).GetAwaiter().GetResult();
    // Nevýhody: blokuje volající vlákno, extra thread switch, horší diagnostika
}
```

### Nito.AsyncEx pro konzolové async kontexty

```csharp
// Nito.AsyncEx.Context – pro synchronní entry points v legacy kódu
// NuGet: Nito.AsyncEx
AsyncContext.Run(async () =>
{
    var data = await StahniAsync();
    Console.WriteLine(data);
});
```

## 3. Kdy použít

- **Async celou cestou** – vždy, pokud je to možné. Každý `await` nahraďte propagací `async Task` výše.
- **`Task.Run().GetAwaiter().GetResult()`** – pouze v synchronních entry points (Main, event handlers v legacy UI bez async support), nikdy uvnitř aplikační logiky.
- **`IAsyncDisposable`** – pro třídy, které potřebují asynchronní cleanup (síťová spojení, soubory s async flush).
- **Factory metoda** – pro asynchronní inicializaci objektu místo async konstruktoru.

## 4. Časté chyby

- ❌ **`task.Result` nebo `task.Wait()` uvnitř async metody** – smíchání způsobí deadlock nebo zbytečné blokování vlákna; vždy `await`.
- ❌ **`async void` pro "fire and forget"** – výjimky jsou neopravitelné; používejte `_ = UlohaAsync().ConfigureAwait(false)` s explicitní error handling.
- ❌ **Synchronní init v konstruktoru přes `.Result`** – konstruktor je volán synchronně; `.Result` na async inicializaci v konstruktoru je deadlock v kontextovém prostředí.
- ❌ **`Thread.Sleep` místo `await Task.Delay`** – `Thread.Sleep` blokuje vlákno; `Task.Delay` ho uvolní.

---

<details>
<summary>Deep dive: jak Task.Run izoluje synchronizační kontext a async entry points</summary>

### Proč Task.Run eliminuje deadlock

`Task.Run` spustí lambda na vlákně z ThreadPoolu, které **nemá synchronizační kontext** (`SynchronizationContext.Current == null`). Continuation async metody pak nečeká na původní vlákno – pokračuje na libovolném vlákně z ThreadPoolu. Volající vlákno (původní) čeká synchronně na `.GetAwaiter().GetResult()`, ale nezpůsobí deadlock, protože continuation nepotřebuje právě toto vlákno.

```
Volající vlákno (UI): Task.Run(lambda).GetAwaiter().GetResult()
  ↓ spustí lambda na ThreadPool vlákně (bez context)
  ↓ lambda awaítuje GetStringAsync
  ↓ po dokončení: continuation na ThreadPoolu (ne na UI vlákně)
  ↓ GetResult se odblokuje
Volající vlákno (UI): pokračuje
```

### Async Main v různých kontextech

```csharp
// .NET 5+ Top-level statements
await SpustAsync();

// .NET 5+ async Main
static async Task Main(string[] args)
{
    await SpustAsync();
}

// Pro unit testy s async: xUnit podporuje async Task testy přímo
[Fact]
public async Task Test_NacteData()
{
    var data = await _sluzba.NactiAsync();
    Assert.NotNull(data);
}
```

### Diagnostika deadlocku

Pokud aplikace "zamrzne" bez výjimky, pravděpodobně jde o deadlock. Jak potvrdit:
1. Attach debugger, break all threads.
2. Thread stacks ukážou vlákno čekající na `.Result` a vlákno čekající na synchronizační kontext.
3. Hledejte: `WaitForSingleObject`, `Monitor.Enter`, `SemaphoreSlim.Wait` ve stacku UI vlákna.

</details>

**Další krok:** [Parallel.ForEachAsync – asynchronní paralelní zpracování](Parallel_ForEachAsync.md)
