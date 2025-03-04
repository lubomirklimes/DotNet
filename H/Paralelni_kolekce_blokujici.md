# Paralelní kolekce v .NET – BlockingCollection a producent-konzument vzor

Když více vláken sdílí jednu frontu dat, nestačí obyčejný `Queue<T>` – přidávání a odebírání nejsou atomické operace a bez synchronizace dojde k poškození dat. `BlockingCollection<T>` tento problém řeší a navíc přidává řízení kapacity a koordinaci mezi producenty a konzumenty.

## 1. Koncept

Vzor producent-konzument rozděluje práci na dvě role: producent data vytváří (nebo přijímá ze sítě, souboru, atd.) a vkládá je do sdílené fronty; konzument je z fronty odebírá a zpracovává. Bez koordinace může producent zahltit frontu a spotřebovat veškerou paměť, nebo konzument marně spinovat a plýtvat CPU na prázdné frontě.

`BlockingCollection<T>` řeší obě situace:
- **Kapacitní limit** (`boundedCapacity`) zablokuje producenta, pokud je fronta plná – přirozený backpressure mechanismus.
- **`Take()`** zablokuje konzumenta, pokud je fronta prázdná – žádné aktivní čekání (busy-wait).
- **`CompleteAdding()`** signalizuje konzumentům, že žádná další data nepřijdou, takže mohou skončit po zpracování zbytku.

Interně `BlockingCollection<T>` obaluje jinou thread-safe kolekci – výchozí je `ConcurrentQueue<T>` (FIFO), ale lze předat i `ConcurrentStack<T>` nebo `ConcurrentBag<T>`.

## 2. Příklad

### Základní producent-konzument

```csharp
// boundedCapacity = 100 znamená: producent se zablokuje, pokud fronta obsahuje 100 položek
var fronta = new BlockingCollection<WorkItem>(boundedCapacity: 100);

// Producent – typicky jeden nebo více Task
var producent = Task.Run(() =>
{
    foreach (var polozka in ZiskejData())
    {
        fronta.Add(polozka); // blokuje, pokud je fronta plná
    }
    fronta.CompleteAdding(); // oznámení: víc dat nepřijde
});

// Konzument – zpracovává položky, dokud fronta není dokončena
var konzument = Task.Run(() =>
{
    // GetConsumingEnumerable() blokující iterátor – vrátí položku nebo čeká; skončí po CompleteAdding
    foreach (var polozka in fronta.GetConsumingEnumerable())
    {
        Zpracuj(polozka);
    }
});

await Task.WhenAll(producent, konzument);
```

### Více producentů a konzumentů

```csharp
var fronta = new BlockingCollection<string>(boundedCapacity: 500);

// Tři producenti paralelně načítají data
var producenti = Enumerable.Range(0, 3).Select(i => Task.Run(() =>
{
    foreach (var radek in File.ReadLines($"soubor_{i}.txt"))
        fronta.Add(radek);
})).ToArray();

// Dva konzumenti paralelně zpracovávají
var konzumenti = Enumerable.Range(0, 2).Select(_ => Task.Run(() =>
{
    foreach (var radek in fronta.GetConsumingEnumerable())
        Zpracuj(radek);
})).ToArray();

// Počkáme na všechny producenty, pak uzavřeme frontu
await Task.WhenAll(producenti);
fronta.CompleteAdding();
await Task.WhenAll(konzumenti);
```

### S podporou zrušení

```csharp
var fronta = new BlockingCollection<WorkItem>(boundedCapacity: 50);
var cts = new CancellationTokenSource();

var konzument = Task.Run(() =>
{
    try
    {
        foreach (var polozka in fronta.GetConsumingEnumerable(cts.Token))
            Zpracuj(polozka);
    }
    catch (OperationCanceledException)
    {
        // Normální ukončení při zrušení
    }
});

// Po 30 sekundách zrušíme
cts.CancelAfter(TimeSpan.FromSeconds(30));
```

## 3. Kdy použít

- **Pipeline zpracování dat** – čtení ze souboru/sítě → parsování → ukládání; každá fáze je spojená `BlockingCollection`.
- **Omezení paralelismu s backpressure** – chcete zpracovávat max. N položek najednou a producentovi říct "počkej".
- **Nahrazení ručních `lock` + `Monitor.Wait/Pulse`** – `BlockingCollection` zapouzdřuje tuto synchronizaci bezpečně.
- **Kanálová architektura** – pokud potřebujete složitější topologii, zvažte `System.Threading.Channels` (viz [Paralelni_zpracovani_Channels.md](Paralelni_zpracovani_Channels.md)).

## 4. Časté chyby

- ❌ **Zapomenutí `CompleteAdding()`** – konzument čeká věčně; `GetConsumingEnumerable()` nikdy neskončí.
- ❌ **Volání `Add()` po `CompleteAdding()`** – vyhodí `InvalidOperationException`. Ujistěte se, že všichni producenti zavolají `CompleteAdding()` koordinovaně (nebo použijte jeden koordinující Task).
- ❌ **Neomezená kapacita v produkci** – bez `boundedCapacity` fronta roste bez omezení; při pomalém konzumentovi dojde paměť.
- ❌ **Použití `Take()` místo `GetConsumingEnumerable()`** – ruční smyčka s `TryTake` je náchylná na race condition při `IsCompleted` check; `GetConsumingEnumerable()` je bezpečnější.

---

<details>
<summary>Deep dive: interní implementace a kdy použít Channels místo BlockingCollection</summary>

### Jak BlockingCollection blokuje bez busy-wait

Interně používá `SemaphoreSlim` pro signalizaci dostupnosti místa (producent) a dostupnosti dat (konzument). `SemaphoreSlim.WaitAsync()` je efektivní – nevytváří nové vlákno, pouze zaregistruje pokračování a vlákno vrátí ThreadPoolu.

### Srovnání BlockingCollection vs System.Threading.Channels

| Vlastnost | `BlockingCollection<T>` | `Channel<T>` |
|-----------|------------------------|-------------|
| API styl | Synchronní (blokující) nebo s CT | Plně async/await |
| Backpressure | `boundedCapacity` | `BoundedChannelOptions` |
| Více čtenářů | Ano | Ano (SingleReader optimalizace) |
| Výkon | Dobrý | Lepší (méně alokací) |
| Zavedeno | .NET Framework 4.0 | .NET Core 2.1 |

Pro nový kód preferujte `System.Threading.Channels` – má async API (žádné blokující čekání), lepší výkon a explicitní `SingleReader`/`SingleWriter` optimalizace.

```csharp
// Moderní ekvivalent s Channels
var channel = Channel.CreateBounded<WorkItem>(new BoundedChannelOptions(100)
{
    FullMode = BoundedChannelFullMode.Wait // backpressure
});

// Producent
await channel.Writer.WriteAsync(item, ct);
channel.Writer.Complete();

// Konzument
await foreach (var item in channel.Reader.ReadAllAsync(ct))
    Zpracuj(item);
```

</details>

**Další krok:** [Paralelní kolekce – ConcurrentQueue, ConcurrentDictionary a další](Paralelni_kolekce_concurrent.md)
