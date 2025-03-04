# Praktické příklady s .NET Parallel Library a PLINQ

`Parallel.For`, `Parallel.ForEach` a PLINQ jsou nástroje pro **CPU-bound paralelismus** – kdy máte výpočetně náročnou práci, která blokuje vlákno. Jsou doplňkem k `async/await`, který řeší I/O-bound čekání.

## 1. Koncept

### Kdy použít TPL vs async/await

| Typ práce | Správný nástroj |
|-----------|----------------|
| Čekání na I/O (síť, disk, DB) | `async/await`, `Task.WhenAll` |
| CPU-bound výpočty (matematika, komprese, kryptografie) | `Parallel.For`, `Parallel.ForEach`, PLINQ |
| Smíšené (načti + zpracuj) | `Parallel.ForEachAsync` nebo `Channel` s oddělenými kroky |

### Parallel.For a Parallel.ForEach

Rozdělí iterace mezi vlákna z ThreadPoolu. Počet souběžných vláken se automaticky přizpůsobuje počtu jader CPU. Vnější smyčka se nevrátí, dokud všechny iterace neskončí.

### PLINQ

Paralelní LINQ – `AsParallel()` přidá paralelizaci na dotaz. Vhodné pro datově náročné transformace. Výsledky mohou přijít v jiném pořadí; `AsOrdered()` pořadí zachová (za cenu synchronizace).

## 2. Příklad

### Parallel.For – CPU-bound výpočet

```csharp
// ✅ CPU-bound: výpočet hash pro každý soubor
var vysledky = new ConcurrentBag<(string soubor, byte[] hash)>();

Parallel.For(0, soubory.Length, i =>
{
    using var sha = SHA256.Create();
    var hash = sha.ComputeHash(File.ReadAllBytes(soubory[i]));
    vysledky.Add((soubory[i], hash));
});
```

### Parallel.ForEach s omezením paralelismu

```csharp
var options = new ParallelOptions
{
    MaxDegreeOfParallelism = Environment.ProcessorCount, // neblokuj více vláken než jader
    CancellationToken = ct
};

Parallel.ForEach(polozky, options, polozka =>
{
    // CPU-bound zpracování – žádné async/await uvnitř
    var vysledek = TransformujData(polozka);
    UlozVysledek(vysledek);
});
```

### Parallel.Invoke – souběžné nezávislé akce

```csharp
// Tři CPU-bound akce spuštěné paralelně
Parallel.Invoke(
    () => KomprimujData(vstup, vystup1),
    () => VypocitejStatistiky(vstup, statistiky),
    () => ValidujSchema(vstup, schema)
);
```

### PLINQ – paralelní LINQ dotaz

```csharp
// ✅ Paralelní filtr + transformace na velké kolekci
var zpracovana = polozky
    .AsParallel()
    .WithDegreeOfParallelism(Environment.ProcessorCount)
    .WithCancellation(ct)
    .Where(p => JeValidni(p))
    .Select(p => Transformuj(p))
    .ToList(); // Materializace – spustí paralelní zpracování

// Zachování pořadí (pomalejší):
var serazena = polozky
    .AsParallel()
    .AsOrdered()
    .Select(p => Transformuj(p))
    .ToList();
```

### ❌ Anti-vzory – blokování ve smyčce

```csharp
// ❌ Task.Delay(1000).Wait() blokuje vlákno ThreadPoolu – plýtvání zdrojem
Parallel.For(0, 10, i =>
{
    Task.Delay(1000).Wait(); // NIKDY: blokuje vlákno
});

// ❌ Task.WaitAll – synchronní blokování
Task.WaitAll(task1, task2, task3); // NIKDY: blokuje volající vlákno

// ✅ Pro I/O-bound paralelismus použijte:
await Task.WhenAll(task1, task2, task3);         // nebo
await Parallel.ForEachAsync(polozky, ct, async (p, t) => await ZpracujAsync(p, t));
```

## 3. Kdy použít

- **`Parallel.For` / `Parallel.ForEach`** – CPU-bound smyčky: transformace obrázků, komprese, kryptografie, matematické výpočty na kolekcích.
- **PLINQ** – LINQ dotazy na velké in-memory kolekce, kde transformace je výpočetně náročná.
- **Ne pro I/O-bound práci** – síťové requesty, čtení souborů, přístup k DB. Tam použijte `async/await` + `Task.WhenAll` nebo `Parallel.ForEachAsync`.

→ Viz [Parallel_ForEachAsync.md](Parallel_ForEachAsync.md) pro moderní async paralelismus.

## 4. Časté chyby

- ❌ **`Task.Delay().Wait()` nebo `task.Result` uvnitř `Parallel.For`** – blokuje vlákno ThreadPoolu; ThreadPool se vyčerpá při větším počtu iterací.
- ❌ **Sdílený mutable stav bez synchronizace** – `List<T>`, `Dictionary` nejsou thread-safe; výsledkem je poškození dat nebo výjimka. Použijte `ConcurrentBag`, `ConcurrentDictionary` nebo lokální proměnné + agregaci.
- ❌ **PLINQ pro malé kolekce** – režie rozdělení práce a synchronizace je větší než zisk; PLINQ má smysl od stovek prvků.
- ❌ **`Parallel.ForEach` pro async lambda** – `Parallel.ForEach` nepodporuje `async` lambdu; async void uvnitř způsobí neodchycené výjimky. Použijte `Parallel.ForEachAsync`.
- ❌ **Zapomenutý `MaxDegreeOfParallelism`** – bez omezení může Parallel vytvořit tolik vláken, kolik má iterací; v ASP.NET Core to sníží propustnost ostatních requestů.

---

<details>
<summary>Deep dive: Partitioner, výkon PLINQ a interakce s ThreadPool</summary>

### Vlastní Partitioner pro nerovnoměrná data

```csharp
// Výchozí dělení: rozsahy (range partitioning) – předpokládá stejně dlouhé iterace
// Pro nerovnoměrná data (různě náročné položky) použijte chunk partitioner:
var partitioner = Partitioner.Create(polozky, loadBalance: true);
Parallel.ForEach(partitioner, options, polozka => Zpracuj(polozka));
```

### PLINQ a výjimky

```csharp
// PLINQ zabalí výjimky do AggregateException
try
{
    var vysledky = polozky.AsParallel().Select(p => Transformuj(p)).ToList();
}
catch (AggregateException aex)
{
    foreach (var ex in aex.InnerExceptions)
        log.LogError(ex, "Zpracování selhalo");
}
```

### ThreadPool saturation

`Parallel.For` implicitně používá ThreadPool. Pokud jsou iterace I/O-bound a blokují vlákna, ThreadPool se saturuje: nové úlohy čekají na uvolnění vlákna. Příznak: CPU je na 0 %, aplikace visí. Diagnostika:

```bash
dotnet-counters monitor --process-id <pid> System.Runtime
# Sledujte: threadpool-queue-length, threadpool-thread-count
```

</details>

**Další krok:** [System.Threading.Channels – producent-konzument pipeline](Paralelni_zpracovani_Channels.md)
