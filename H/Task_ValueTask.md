# Správné použití `Task` a `ValueTask`

`Task<T>` je výchozí návratový typ pro asynchronní metody v .NET. `ValueTask<T>` je optimalizace pro metody, které se **obvykle dokončí synchronně** – vrátí výsledek okamžitě bez skutečného čekání.

## 1. Koncept

### Task\<T\>

`Task<T>` je referenční typ – vždy se alokuje na haldě. Je vhodný pro operace, které skutečně čekají (I/O, síť, DB). Runtime může cachovat hotové tasky (`Task.CompletedTask`, `Task.FromResult`) pro eliminaci alokace.

### ValueTask\<T\>

`ValueTask<T>` je hodnotový typ (struct). Interně drží buď přímo výsledek (bez alokace), nebo odkaz na `Task<T>`. Eliminuje alokaci v **hot path**, kde operace typicky vrátí cachovaný výsledek nebo se dokončí synchronně (např. cache hit).

**Klíčové omezení:** `ValueTask` smí být awaited **pouze jednou** a nesmí být awaited po dokončení jiného `await`. Porušení vede k nedefinovanému chování.

### Kdy co použít

| Situace | Typ |
|---------|-----|
| Obecné async metody (I/O, síť, DB) | `Task<T>` |
| Metody s rychlou synchronní cestou (cache, výsledek z paměti) | `ValueTask<T>` |
| Rozhraní pro volné spárování (Interface members) | `ValueTask<T>` (flexibilnější pro implementace) |
| Metody volané z `async` smyčky tisíckrát za sekundu | `ValueTask<T>` |

## 2. Příklad

### Task.FromResult – cachovaný výsledek bez alokace

```csharp
// Task.FromResult je levnější než new Task – runtime cachuje common hodnoty
public Task<bool> JeAktivniAsync(int id)
{
    if (_cache.TryGetValue(id, out bool cached))
        return Task.FromResult(cached); // Žádná alokace pro bool true/false

    return NactiZDatabázeAsync(id);
}
```

### ValueTask – synchronní hot path

```csharp
public interface ICache<T>
{
    ValueTask<T?> GetAsync(string key, CancellationToken ct = default);
}

public class InMemoryCache<T> : ICache<T>
{
    private readonly Dictionary<string, T> _store = new();

    // ✅ Cache hit → synchronní návrat, žádná alokace
    // Cache miss → skutečná async operace
    public ValueTask<T?> GetAsync(string key, CancellationToken ct = default)
    {
        if (_store.TryGetValue(key, out var val))
            return ValueTask.FromResult<T?>(val); // synchronní, bez alokace

        return new ValueTask<T?>(NactiAsync(key, ct)); // async cesta
    }

    private async Task<T?> NactiAsync(string key, CancellationToken ct) { /* ... */ return default; }
}
```

### ❌ Chybné použití ValueTask

```csharp
var vt = GetValueAsync();

await vt;           // ✅ první await
await vt;           // ❌ druhý await – nedefinované chování!

var t = vt.AsTask(); // ❌ AsTask() po awaitu – nedefinované chování
```

### IValueTaskSource – pokročilá optimalizace

```csharp
// Pro ultra-low-latency scénáře (např. Socket, Pipe) implementujte IValueTaskSource
// – umožňuje recyklovat state machine bez alokace Task
// Běžně implementováno v System.IO.Pipelines (PipeReader.ReadAsync vrací ValueTask)
int bytesRead = await pipeReader.ReadAsync(ct); // ValueTask bez alokace
```

## 3. Kdy použít

- **`Task<T>`** – výchozí volba pro vše: async controllery, service metody, background workery. Jednoduché, kompatibilní s celým ekosystémem.
- **`ValueTask<T>`** – použijte cíleně jako optimalizaci, kdy profiler ukáže, že alokace `Task` je měřitelný problém. Typické: implementace rozhraní pro cache, `IAsyncEnumerator`, `Channel`, I/O pipeline.
- **`Task` (non-generic)** – pro metody bez výsledku; alternativou je `ValueTask` (non-generic) pro stejné optimalizační případy.

## 4. Časté chyby

- ❌ **`ValueTask` awaited vícekrát** – jedinou bezpečnou operací po prvním `await` je zahodit `ValueTask`. Pokud potřebujete sdílet výsledek, zavolejte `.AsTask()` **před** prvním awaitem.
- ❌ **`ValueTask` uložený do proměnné a předaný jinam** – může být awaited více místy; to je nedefinované chování. Použijte `Task`.
- ❌ **Vlastní `async` metoda vrací `ValueTask` ale vždy je async** – pokud metoda vždy čeká, `ValueTask` nepomůže (vždy se alokuje state machine). Profiler ukáže 0% synchronních dokončení.
- ❌ **`task.Result` nebo `task.Wait()` pro získání výsledku** – blokuje vlákno. Vždy `await task`.

---

<details>
<summary>Deep dive: IValueTaskSource, ManualResetValueTaskSourceCore a benchmarky</summary>

### Proč ValueTask nestačí pro recyklaci

Základní `ValueTask<T>` eliminuje alokaci při synchronním dokončení, ale při async dokončení stále vznikne `Task<T>` (nebo ekvivalentní state machine). `IValueTaskSource<T>` umožňuje **recyklovat** state object:

```csharp
// System.Threading.Channels.Channel interně používá ManualResetValueTaskSourceCore<T>
// pro recyklaci bez alokací i v async cestě
var channel = Channel.CreateUnbounded<int>();
// ReadAsync vrací ValueTask<T> bez alokace Task i při čekání
int item = await channel.Reader.ReadAsync(ct);
```

### Benchmarky (BenchmarkDotNet)

| Metoda | Čas | Alokace |
|--------|-----|---------|
| `Task.FromResult(42)` | 15 ns | 56 B |
| `ValueTask.FromResult(42)` | 2 ns | 0 B |
| `async Task` (synchronní cesta) | 50 ns | 120 B |
| `async ValueTask` (synchronní cesta) | 5 ns | 0 B |

Rozdíl je znatelný pouze při **milionech volání za sekundu** – neoptimalizujte předčasně.

### Doporučení pro rozhraní knihoven

```csharp
// Rozhraní knihovny: preferujte ValueTask pro maximální flexibilitu implementace
public interface IRepository<T>
{
    ValueTask<T?> GetByIdAsync(int id, CancellationToken ct = default);
    ValueTask<IReadOnlyList<T>> GetAllAsync(CancellationToken ct = default);
}
```

Implementace může vrátit synchronně cachovaná data bez alokace, nebo delegovat na skutečný async zdroj.

</details>

**Další krok:** [CancellationToken a řízení zrušení úloh](CancellationToken.md)
