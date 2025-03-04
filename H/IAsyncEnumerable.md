# `IAsyncEnumerable<T>` a asynchronní streamování dat

`IAsyncEnumerable<T>` umožňuje **asynchronní iteraci** – prvky přicházejí postupně a každé získání dalšího prvku může čekat na I/O. Na rozdíl od `Task<IEnumerable<T>>` nečeká na načtení všech dat najednou.

## 1. Koncept

Standardní `IEnumerable<T>` předpokládá synchronní přístup k prvkům. `IAsyncEnumerable<T>` (zavedeno v C# 8 / .NET Core 3.0) přidává `await` do iterace: `await foreach` čeká na každý prvek asynchronně, aniž blokuje vlákno.

Kompilátor transformuje `async` metodu s `yield return` na state machine implementující `IAsyncEnumerable<T>`. Zrušení se předává přes `[EnumeratorCancellation] CancellationToken` a `WithCancellation()`.

### IAsyncEnumerable\<T\> vs Task\<IEnumerable\<T\>\>

| | `Task<IEnumerable<T>>` | `IAsyncEnumerable<T>` |
|---|---|---|
| Data načtena | Všechna najednou | Postupně (streaming) |
| Paměť | Vše v paměti | Po jednom prvku |
| První prvek k dispozici | Až po načtení všeho | Okamžitě |
| Vhodné pro | Malé datové sady | Velké sady, DB kurzory, streamy |
| Zrušení | Před zahájením | Kdykoli během iterace |

## 2. Příklad

### Základní asynchronní iterátor

```csharp
async IAsyncEnumerable<int> GenerujCislaAsync(
    [EnumeratorCancellation] CancellationToken ct = default)
{
    for (int i = 0; i < 100; i++)
    {
        ct.ThrowIfCancellationRequested();
        await Task.Delay(10, ct); // simulace I/O
        yield return i;
    }
}

await foreach (int cislo in GenerujCislaAsync().WithCancellation(cts.Token))
    Console.WriteLine(cislo);
```

### Čtení řádků ze souboru

```csharp
async IAsyncEnumerable<string> CtiRadkyAsync(
    string cesta,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    using var reader = new StreamReader(cesta);
    while (await reader.ReadLineAsync(ct) is { } radek)
        yield return radek;
}

await foreach (var radek in CtiRadkyAsync("data.txt"))
    await ZpracujRadekAsync(radek);
```

### Čtení z databáze (EF Core)

```csharp
// AsAsyncEnumerable() načítá řádky postupně přes databázový kurzor
// – nealokuje celou tabulku do paměti najednou
await foreach (var produkt in _db.Produkty
    .Where(p => p.Aktivni)
    .AsAsyncEnumerable()
    .WithCancellation(ct))
{
    await ZpracujProduktAsync(produkt, ct);
}
```

### Streamování přes Minimal API / ASP.NET Core

```csharp
// ASP.NET Core automaticky streamuje JSON odpověď na klienta
app.MapGet("/produkty/stream", (IProduktRepository repo, CancellationToken ct) =>
    repo.GetAllAsync(ct)); // vrací IAsyncEnumerable<Produkt>

// Na klientovi (HttpClient):
var stream = httpClient.GetFromJsonAsAsyncEnumerable<Produkt>("/produkty/stream", ct);
await foreach (var produkt in stream!)
    Zobraz(produkt);
```

### ConfigureAwait v await foreach

```csharp
// V knihovním kódu použijte ConfigureAwait(false)
await foreach (var item in ZdrojAsync().ConfigureAwait(false))
    Zpracuj(item);
```

## 3. Kdy použít

- **Velké datové sady z DB** – EF Core `AsAsyncEnumerable()` místo `ToListAsync()` když zpracováváte jednotlivé záznamy.
- **Streamování API odpovědí** – `/stream` endpoint v Minimal API nebo gRPC server streaming.
- **Čtení velkých souborů** – po řádcích nebo blocích bez načtení celého souboru do paměti.
- **Real-time data** – události z fronty (RabbitMQ, Kafka), telemetrie, log tail.
- **Generátory s I/O** – když generování každého dalšího prvku vyžaduje await (stránkování API, lazy loading).

**Nepoužívejte** pro malé kolekce, kde `Task<List<T>>` je jednodušší, nebo kde potřebujete náhodný přístup k prvkům.

## 4. Časté chyby

- ❌ **Zapomenutý `[EnumeratorCancellation]`** – `WithCancellation(ct)` předá token do iterátoru pouze pokud je parametr označen `[EnumeratorCancellation]`; bez atributu token nemá efekt.
- ❌ **`ToListAsync()` místo `AsAsyncEnumerable()`** – `ToListAsync()` načte vše do paměti; při milionech záznamů způsobí OOM nebo tlak na GC.
- ❌ **`await foreach` bez `WithCancellation`** – iterace pokračuje i po zrušení požadavku; vždy předejte `CancellationToken` volajícího.
- ❌ **Výjimky uvnitř iterátoru bez ošetření** – výjimka přerušní `await foreach` s výjimkou; `finally` blok v iterátoru se zavolá (Dispose); zajistěte správné uvolnění zdrojů.
- ❌ **Zanechání nezkonzumovaného `IAsyncEnumerable`** – pokud nedoiterujete do konce, volá se `DisposeAsync` na enumerátoru; ujistěte se, že iterátor správně implementuje `finally` (kompilátor to zajistí automaticky u `yield return` metod).

---

<details>
<summary>Deep dive: IAsyncEnumerator, ConfigureAwait a backpressure</summary>

### Jak funguje IAsyncEnumerator interně

`await foreach` kompiluje na:

```csharp
var enumerator = source.GetAsyncEnumerator(ct);
try
{
    while (await enumerator.MoveNextAsync())
        Zpracuj(enumerator.Current);
}
finally
{
    await enumerator.DisposeAsync();
}
```

`MoveNextAsync()` vrací `ValueTask<bool>` – u rychlých generátorů (data v paměti) dokončí synchronně bez alokace.

### Backpressure s Channel

`IAsyncEnumerable` samo o sobě backpressure neimplementuje – konzument řídí tempo (`MoveNextAsync` se volá, až je konzument připraven). Pro explicitní backpressure kombinujte s `Channel<T>`:

```csharp
// Producent naplňuje kanál; konzument čte přes IAsyncEnumerable
var channel = Channel.CreateBounded<Polozka>(capacity: 100);
// channel.Reader.ReadAllAsync() vrací IAsyncEnumerable<Polozka>
await foreach (var item in channel.Reader.ReadAllAsync(ct))
    await ZpracujAsync(item, ct);
```

### LINQ nad IAsyncEnumerable

```csharp
// System.Linq.Async (NuGet) přidává Where, Select, Take aj.
var prvnich10 = await zdroj
    .WhereAwait(async p => await JeValidniAsync(p))
    .Take(10)
    .ToListAsync(ct);
```

</details>

**Další krok:** [TaskCompletionSource – pokročilé použití](TaskCompletionSource.md)
