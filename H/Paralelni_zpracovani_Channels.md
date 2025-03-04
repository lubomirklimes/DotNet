# Paralelní zpracování s `System.Threading.Channels`

`Channel<T>` je high-performance, asynchronní **producent-konzument fronta**. Na rozdíl od `BlockingCollection<T>` (která blokuje vlákno) podporuje plně asynchronní čtení i zápis – ideální pro pipeline zpracování dat v ASP.NET Core, BackgroundService a mikroservisách.

## 1. Koncept

### Channel vs BlockingCollection

| | `Channel<T>` | `BlockingCollection<T>` |
|---|---|---|
| Čekání | Asynchronní (`await`) | Synchronní (blokuje vlákno) |
| Vhodné pro | ASP.NET Core, async pipelines | Starší kód, konzolové aplikace |
| Backpressure | `BoundedChannel` s `await WriteAsync` | `BoundingCapacity` + blokování |
| Výkon | Vyšší (lock-free pro unbounded) | Nižší (Monitor.Wait) |

→ Viz [Paralelni_kolekce_blokujici.md](Paralelni_kolekce_blokujici.md) pro `BlockingCollection` vzor.

### Bounded vs Unbounded

- **`Channel.CreateUnbounded<T>()`** – bez limitu; producent nikdy nečeká. Riziko: nekontrolovaný růst paměti.
- **`Channel.CreateBounded<T>(kapacita)`** – producent čeká (nebo dostane chybu), pokud je kanál plný. Implementuje přirozenou **backpressure**.

## 2. Příklad

### Základní producent-konzument

```csharp
var channel = Channel.CreateBounded<int>(capacity: 100);

// Producent
async Task ProducentAsync(CancellationToken ct)
{
    for (int i = 0; i < 1000; i++)
    {
        await channel.Writer.WriteAsync(i, ct); // čeká, pokud je kanál plný
    }
    channel.Writer.Complete(); // signalizuje konec – ReadAllAsync se ukončí
}

// Konzument
async Task KonzumentAsync(CancellationToken ct)
{
    await foreach (var item in channel.Reader.ReadAllAsync(ct))
    {
        await ZpracujAsync(item, ct);
    }
}

await Task.WhenAll(ProducentAsync(ct), KonzumentAsync(ct));
```

### Více konzumentů – paralelní zpracování

```csharp
var channel = Channel.CreateBounded<WorkItem>(capacity: 200);

// Jeden producent
var producent = Task.Run(() => NaplnKanalAsync(channel.Writer, ct), ct);

// Tři paralelní konzumenti – každý čte z téhož kanálu
var konzumenti = Enumerable.Range(0, 3)
    .Select(_ => Task.Run(() => KonzumentAsync(channel.Reader, ct), ct))
    .ToArray();

await producent;
await Task.WhenAll(konzumenti);

async Task KonzumentAsync(ChannelReader<WorkItem> reader, CancellationToken ct)
{
    await foreach (var item in reader.ReadAllAsync(ct))
        await ZpracujAsync(item, ct);
}
```

### Pipeline – více kanálů za sebou

```csharp
// Fáze 1: stáhni → Fáze 2: parsuj → Fáze 3: ulož
var stahovani  = Channel.CreateBounded<string>(50);
var parsovani  = Channel.CreateBounded<ParsedData>(50);

var f1 = StahujAsync(urls, stahovani.Writer, ct);
var f2 = ParsujAsync(stahovani.Reader, parsovani.Writer, ct);
var f3 = UkladejAsync(parsovani.Reader, ct);

await Task.WhenAll(f1, f2, f3);
```

### Bounded s BoundedChannelFullMode

```csharp
// Místo čekání producenta – zahoď nejstarší položku (pro real-time telemetrii)
var channel = Channel.CreateBounded<Telemetrie>(new BoundedChannelOptions(1000)
{
    FullMode = BoundedChannelFullMode.DropOldest,
    SingleReader = true,   // optimalizace: jen jeden konzument
    SingleWriter = false
});
```

## 3. Kdy použít

- **BackgroundService s frontou** – API přijme request, hodí do kanálu, background worker zpracuje asynchronně.
- **Pipeline zpracování** – více fází (stáhnout → transformovat → uložit) s přirozenou backpressure mezi nimi.
- **Fan-out** – jeden producent, více konzumentů pro paralelní zpracování.
- **Real-time data** – telemetrie, log aggregace, event streaming kde nechcete blokovat producenta.

## 4. Časté chyby

- ❌ **`Writer.Complete()` zapomenuto** – `ReadAllAsync` nikdy neskončí; konzument čeká věčně.
- ❌ **`UnboundedChannel` bez backpressure** – producent je rychlejší než konzument → neomezený růst paměti. Vždy zvažte `BoundedChannel`.
- ❌ **`TryWrite` místo `WriteAsync` při `BoundedChannel`** – `TryWrite` vrátí `false` pokud je kanál plný a zahodí data. Použijte `WriteAsync` pro backpressure nebo vědomě ošetřte `false`.
- ❌ **Sdílení `ChannelWriter`/`ChannelReader` napříč více producenty bez `SingleWriter = false`** – výchozí nastavení je bezpečné pro více producentů, ale nastavení `SingleWriter = true` dá runtime prostor k optimalizaci.
- ❌ **Catch výjimky od producenta izolovaně** – pokud producent vyhodí výjimku bez `Writer.Complete(ex)`, konzument čeká dál. Vždy: `channel.Writer.Complete(výjimka)` v `catch` bloku.

---

<details>
<summary>Deep dive: Channel internals, výkon vs BlockingCollection a diagnostika</summary>

### Interní implementace

`UnboundedChannel<T>` používá lock-free `ConcurrentQueue<T>` pro Enqueue a `ManualResetValueTaskSourceCore` pro asynchronní čekání konzumentů bez alokace `Task`. Bounded verze používá `DequeOfT` (interní) s semaforem pro backpressure.

### Výkonové srovnání (BenchmarkDotNet, .NET 8)

| Scénář | Channel | BlockingCollection |
|--------|---------|--------------------|
| 1M items, 1 producent, 1 konzument | ~200 ms | ~800 ms |
| Alokace na item | ~0 B | ~40 B |
| CPU při čekání | 0 % (async) | vlákno blokováno |

### Diagnostika

Pokud konzument nestíhá, sledujte délku fronty:

```csharp
// ChannelReader.Count (pouze odhad u Unbounded)
log.LogInformation("Fronta: {Count}", channel.Reader.Count);
```

Pro přesné metriky integrujte s `System.Diagnostics.Metrics`:

```csharp
var meter = new Meter("MojeAplikace");
var gauge = meter.CreateObservableGauge("channel.queue.length",
    () => channel.Reader.Count);
```

</details>

**Další krok:** [Paralelní kolekce – BlockingCollection a producent-konzument vzor](Paralelni_kolekce_blokujici.md)
