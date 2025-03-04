# Parallel.ForEachAsync – asynchronní paralelní zpracování kolekcí

`Parallel.ForEachAsync` je .NET 6 API pro paralelní zpracování kolekcí, kde je každá položka zpracována asynchronní metodou. Zaplňuje mezeru mezi synchronním `Parallel.ForEach` (CPU-bound) a `Task.WhenAll` (bez omezení paralelismu).

## 1. Koncept

Před `Parallel.ForEachAsync` jste měli dvě možnosti:

1. `Task.WhenAll(items.Select(item => ProcessAsync(item)))` – spustí **všechny** tasky najednou. Pokud máte 10 000 položek, spustíte 10 000 souběžných operací, přetížíte databázi nebo síť.
2. Sekvenční `foreach` s `await` – zpracovává po jedné, pomalé.

`Parallel.ForEachAsync` kombinuje obojí: zpracovává asynchronně, ale s nastaveným maximálním počtem souběžných operací (`MaxDegreeOfParallelism`). Je to jako thread pool pro async operace.

## 2. Příklad

### Základní použití

```csharp
var polozky = Enumerable.Range(1, 1000).ToList();

await Parallel.ForEachAsync(
    polozky,
    new ParallelOptions
    {
        MaxDegreeOfParallelism = 10,  // Max 10 souběžných operací
        CancellationToken = ct
    },
    async (polozka, token) =>
    {
        await ZpracujPolozku(polozka, token);
    });
```

### Zpracování souborů s omezením I/O

```csharp
// Zpracování 500 souborů, max 20 souběžně (I/O bound)
var soubory = Directory.GetFiles(adresare, "*.csv");

await Parallel.ForEachAsync(
    soubory,
    new ParallelOptions { MaxDegreeOfParallelism = 20, CancellationToken = ct },
    async (soubor, token) =>
    {
        var radky = await File.ReadAllLinesAsync(soubor, token);
        await _repo.UlozRadkyAsync(radky, token);
        log.LogInformation("Zpracován: {Soubor}", Path.GetFileName(soubor));
    });
```

### Zpracování s výsledky (sběr výstupů)

```csharp
// Parallel.ForEachAsync nevrací výsledky přímo – použijte ConcurrentBag nebo Channel
var vysledky = new ConcurrentBag<Vysledek>();

await Parallel.ForEachAsync(
    vstupniData,
    new ParallelOptions { MaxDegreeOfParallelism = 8, CancellationToken = ct },
    async (polozka, token) =>
    {
        var vysledek = await AnalyzujAsync(polozka, token);
        vysledky.Add(vysledek);
    });

return vysledky.ToList();
```

### Srovnání přístupů

```csharp
var polozky = Enumerable.Range(1, 10_000).ToList();

// ❌ Sekvenční – 10 000 * 100ms = ~17 minut
foreach (var p in polozky)
    await ZpracujAsync(p, ct);

// ⚠️ WhenAll bez omezení – 10 000 souběžných; může přetížit databázi
await Task.WhenAll(polozky.Select(p => ZpracujAsync(p, ct)));

// ✅ Parallel.ForEachAsync – max 20 souběžných; ~50 sekund, kontrolovaný tlak
await Parallel.ForEachAsync(polozky,
    new ParallelOptions { MaxDegreeOfParallelism = 20, CancellationToken = ct },
    async (p, t) => await ZpracujAsync(p, t));
```

## 3. Kdy použít

- **Dávkové zpracování s I/O** – odesílání e-mailů, volání externího API, zápis do databáze, zpracování souborů.
- **Když `Task.WhenAll` přetěžuje backend** – databáze, API nebo síť mají kapacitní limit; `MaxDegreeOfParallelism` ho respektuje.
- **Migrace dat** – importy, transformace, re-indexace.
- **Alternativa k `PLINQ`** – PLINQ je pro CPU-bound operace; `Parallel.ForEachAsync` pro I/O-bound asynchronní operace.

## 4. Časté chyby

- ❌ **Příliš vysoký `MaxDegreeOfParallelism`** – 100+ pro databázové operace způsobí connection pool exhaustion. Pro databázi 5–20, pro HTTP API záleží na rate limitu.
- ❌ **Zachycení výjimky uvnitř lambda bez re-throw** – pokud spolknete výjimku, `Parallel.ForEachAsync` skončí bez chyby; ztratíte informaci o selhání.
- ❌ **Sdílený stav bez synchronizace** – přidávání do `List<T>` z více vláken způsobí poškození dat; používejte `ConcurrentBag<T>` nebo `Interlocked`.
- ❌ **`MaxDegreeOfParallelism = -1`** – `–1` znamená bez omezení, stejné jako `Task.WhenAll`. Pro skutečně neomezený paralelismus je `Task.WhenAll` čitelnější.

---

<details>
<summary>Deep dive: srovnání s PLINQ, Channels a SemaphoreSlim</summary>

### Parallel.ForEachAsync vs SemaphoreSlim

Před .NET 6 byl standardní vzor:

```csharp
using var semafory = new SemaphoreSlim(20); // Max 20 souběžných

var tasky = polozky.Select(async polozka =>
{
    await semafory.WaitAsync(ct);
    try { await ZpracujAsync(polozka, ct); }
    finally { semafory.Release(); }
});

await Task.WhenAll(tasky);
```

`Parallel.ForEachAsync` dělá totéž, ale je čitelnější a efektivnější (méně alokací pro state machines).

### Parallel.ForEachAsync vs System.Threading.Channels

Channels jsou vhodné pro **pipeline architekturu** (producent–konzument s více fázemi). `Parallel.ForEachAsync` je vhodné pro jednoduchou paralelizaci jedné operace bez pipeline.

```
Parallel.ForEachAsync: vstupní kolekce → async zpracování → konec
Channels:              producent → fronta → konzument fáze 1 → fronta → konzument fáze 2 → ...
```

### Výkon a alokace

`Parallel.ForEachAsync` interně používá `SemaphoreSlim` a spravuje frontu práce efektivněji než ruční `Task.WhenAll` s lambdami. Pro 10 000+ položek je méně náročné na paměť, protože nespouští všechny state machines najednou.

</details>

**Další krok:** [TPL a PLINQ – paralelní zpracování CPU-bound úloh](TPL_PLINQ.md)
