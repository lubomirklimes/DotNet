# Minimalizace přepínání vláken a ThreadPool Tuning

Přepínání vláken (context switch) má nenulovou cenu. V ASP.NET Core s tisíci souběžných requestů může špatná konfigurace ThreadPoolu nebo zbytečné blokování vláken znatelně snížit propustnost a zvýšit latenci.

## 1. Koncept

### Co je context switch

Operační systém pozastaví vlákno, uloží jeho stav (registry, zásobník pointer) a přepne na jiné. Jeden context switch trvá 1–10 µs. Při tisících vláknech to znamená výraznou CPU zátěž bez užitečné práce.

### ThreadPool v .NET

.NET spravuje vlákna přes `ThreadPool`. Nové vlákno se přidává pomalu (1 vlákno/sekunda po dosažení minima) – aby se zamezilo přidávání vláken při krátkodobých špičkách. `async/await` uvolňuje vlákna zpět do ThreadPoolu při čekání, čímž dramaticky snižuje počet potřebných vláken.

### Proč async/await snižuje přepínání

Synchronní blokování (`Thread.Sleep`, `Task.Wait`, `task.Result`) drží vlákno nečinné. `await` vlákno uvolní – ThreadPool ho přidělí jinému requestu. Výsledek: místo 1 vlákno / 1 request stačí desítky vláken pro tisíce requestů.

→ Viz [Sprava_pameti.md](Sprava_pameti.md) pro GC vliv na výkon.  
→ Viz [Profilovani_aplikaci.md](Profilovani_aplikaci.md) pro diagnostiku ThreadPool nasycení.

## 2. Příklad

### Diagnostika – sledování ThreadPool

```csharp
// Programatické sledování stavu ThreadPoolu
ThreadPool.GetMinThreads(out int minWorker, out int minIO);
ThreadPool.GetMaxThreads(out int maxWorker, out int maxIO);
ThreadPool.GetAvailableThreads(out int availWorker, out int availIO);

Console.WriteLine($"Worker: min={minWorker}, max={maxWorker}, volná={availWorker}");
Console.WriteLine($"IO:     min={minIO}, max={maxIO}, volná={availIO}");

// Živé metriky přes dotnet-counters:
// dotnet-counters monitor --process-id <pid> System.Runtime
// Sledujte: threadpool-queue-length, threadpool-thread-count
```

### SetMinThreads – eliminace zpoždění při špičce

```csharp
// Výchozí min = počet CPU jader. ASP.NET Core pod zátěží
// musí čekat na pomalé přidávání vláken.
// Nastavit v Program.cs při startu aplikace:
ThreadPool.SetMinThreads(
    workerThreads: Environment.ProcessorCount * 4,
    completionPortThreads: Environment.ProcessorCount * 4
);
```

### ❌ Anti-vzory – zbytečné blokování

```csharp
// ❌ .Result a .Wait() blokují vlákno ThreadPoolu
var data = GetDataAsync().Result;     // NIKDY v ASP.NET Core
var data = GetDataAsync().GetAwaiter().GetResult(); // NIKDY – deadlock riziko

// ❌ Thread.Sleep blokuje vlákno
Thread.Sleep(1000); // NIKDY – použijte await Task.Delay(1000, ct)

// ✅ Vždy async/await
var data = await GetDataAsync(ct);
await Task.Delay(1000, ct);
```

### TaskScheduler – omezení souběžnosti

```csharp
// Omezení počtu souběžných CPU-bound úloh (neblokujícím způsobem)
// Pro CPU-bound workloady s omezením použijte SemaphoreSlim:
private static readonly SemaphoreSlim _limiter = new(initialCount: 4, maxCount: 4);

public async Task ZpracujAsync(WorkItem item, CancellationToken ct)
{
    await _limiter.WaitAsync(ct);
    try
    {
        await Task.Run(() => HeavyCpuWork(item), ct);
    }
    finally
    {
        _limiter.Release();
    }
}
```

### Parallel.ForEachAsync – správný async paralelismus

```csharp
// Místo ThreadPool.QueueUserWorkItem nebo Task.Factory.StartNew pro async práci:
await Parallel.ForEachAsync(
    polozky,
    new ParallelOptions { MaxDegreeOfParallelism = 8, CancellationToken = ct },
    async (polozka, token) => await ZpracujAsync(polozka, token)
);
```

## 3. Kdy se věnovat ThreadPool tuningu

- Aplikace má high latency P99 i při nízkém CPU (příznak ThreadPool nasycení).
- `threadpool-queue-length` metrika roste – vlákna nestíhají zpracovávat požadavky.
- Startup aplikace je pomalý pod zátěží – ThreadPool nestačí rychle přidat vlákna.
- Máte synchronní kód (třetí strany, legacy) integrovaný do async pipeline.

## 4. Časté chyby

- ❌ **`task.Result` nebo `.Wait()` v ASP.NET Core** – blokuje vlákno ThreadPoolu; při vysoké zátěži způsobí ThreadPool nasycení a deadlock.
- ❌ **`Thread.Sleep` místo `await Task.Delay`** – drží vlákno nečinné; vlákno nelze přidělit jinému requestu.
- ❌ **Příliš vysoký `SetMaxThreads`** – více vláken = více context switchů; nad počtem CPU jader jsou vlákna kontraproduktivní pro CPU-bound práci.
- ❌ **`new Thread(...)` pro každý request** – vytvoření vlákna trvá ~100 µs a spotřebuje ~1 MB zásobník. Vždy ThreadPool nebo async.
- ❌ **Spoléhat na výchozí `SetMinThreads` pro burst traffic** – výchozí minimum je nízké; při náhlé špičce trvá sekundy, než ThreadPool přidá vlákna.

---

<details>
<summary>Deep dive: ThreadPool heuristika, Hill Climbing a IOCP</summary>

### Hill Climbing algoritmus

.NET ThreadPool používá Hill Climbing (HC) algoritmus pro automatické nastavení počtu worker vláken:
- Měří propustnost (completed items/s)
- Experimentuje s přidáváním/odebíráním vláken
- Konverguje k optimálnímu počtu pro aktuální zátěž

HC reaguje pomalu (sekundy) – proto `SetMinThreads` pomáhá při startup a burst scénářích.

### IOCP – I/O Completion Ports

Na Windows používá ThreadPool IOCP pro asynchronní I/O. Dokončení I/O operace (network, disk) nespouští nové vlákno – existující vlákno z IOCP poolu dostane callback. Proto `completionPortThreads` v `SetMinThreads` ovlivňuje latenci sítě a disku.

### Diagnostika pomocí dotnet-trace

```bash
# Zachycení ThreadPool událostí
dotnet-trace collect --process-id <pid> \
  --providers Microsoft-Windows-DotNETRuntime:0x10000:4

# Analyzujte v PerfView: ThreadPool starvation, injection events
```

Hledejte: `ThreadPoolWorkerThreadAdjustmentAdjustment` events kde `reason = Starvation`.

</details>

**Další krok:** [Pokročilé techniky s async a await](Async_await_pokrocile.md)
