# Správa paměti a Garbage Collection v .NET

.NET spravuje paměť automaticky přes Garbage Collector (GC). Pro většinu aplikací to funguje transparentně. Problémy nastávají, když generujete příliš mnoho krátkodobých objektů, alokujete velké bloky paměti, nebo udržujete reference déle, než je nutné – GC pak pracuje zbytečně tvrdě a způsobuje pauzy i degradaci throughputu.

## 1. Koncept

### Generační model

GC v .NET pracuje se třemi generacemi:

| Generace | Kdo tam patří | Jak často se sbírá | Jak dlouho trvá |
|----------|--------------|-------------------|-----------------|
| **Gen 0** | Nové, krátkodobé objekty | Velmi často (stovky za minutu) | Mikrosekundy |
| **Gen 1** | Objekty přeživší Gen 0 | Méně často | Milisekundy |
| **Gen 2** | Dlouhodobé objekty | Zřídka | Desítky ms nebo víc |
| **LOH** | Objekty > 85 000 B | Spolu s Gen 2 | Nekompaktuje se (!) |

Cíl je, aby většina objektů zemřela v Gen 0 (krátce po alokaci). Pokud objekty přežívají a dostávají se do Gen 2, GC sbírky jsou drahé. LOH (Large Object Heap) je zvlášť problematický – nekompaktuje se automaticky, takže se může fragmentovat.

### Server GC vs Workstation GC

- **Workstation GC** (výchozí pro desktopy): minimalizuje pauzy, GC běží na jednom vlákně.
- **Server GC** (výchozí pro ASP.NET Core): každé CPU jádro dostane vlastní haldu a GC vlákno; vyšší throughput, ale vyšší paměťová spotřeba.

## 2. Příklad

### Identifikace problémů – co sledovat

```csharp
// Programatický přístup ke GC statistikám
var gcInfo = GC.GetGCMemoryInfo();
Console.WriteLine($"Gen 0: {GC.CollectionCount(0)} sbírek");
Console.WriteLine($"Gen 1: {GC.CollectionCount(1)} sbírek");
Console.WriteLine($"Gen 2: {GC.CollectionCount(2)} sbírek");
Console.WriteLine($"Heap size: {gcInfo.HeapSizeBytes / 1024 / 1024} MB");
Console.WriteLine($"Fragmentace: {gcInfo.FragmentedBytes / 1024 / 1024} MB");
```

### Snižování alokací – pooling

Pro dočasné buffery (64 KB, 1 MB) použijte `ArrayPool<byte>.Shared` místo `new byte[n]` – vrátí pole zpět do poolu a eliminuje alokaci při dalším requestu. → Viz [ArrayPool.md](ArrayPool.md) pro kompletní příklady a Rent/Return vzor.

Pro práci s přečtenými bajty bez kopírování kombinujte s `Span<T>` → viz [Span_Memory.md](Span_Memory.md).

### Správné nakládání s IDisposable

```csharp
// ❌ Finalizer jako záchranná síť je pozdě – objekt přežije do Gen 1 nebo 2
public class Zdroj
{
    private readonly NativniHandle _handle;

    ~Zdroj() { _handle.Free(); } // Finalizer = drahé; GC musí objekt přežít navíc
}

// ✅ IDisposable + using = deterministické uvolnění, žádný finalizer
public class Zdroj : IDisposable
{
    private readonly NativniHandle _handle;
    private bool _disposed;

    public void Dispose()
    {
        if (_disposed) return;
        _handle.Free();
        _disposed = true;
        GC.SuppressFinalize(this); // Říká GC: finalizer nevolat
    }
}

// Vždy using – Dispose se zavolá i při výjimce
using var zdroj = new Zdroj();
```

### LOH – vyhýbejte se velkým krátkodobým alokacím

```csharp
// ❌ Každé volání alokuje 1 MB na LOH → fragmentace
public string SerializujAsync(List<Polozka> polozky)
{
    return JsonSerializer.Serialize(polozky); // velký string může jít na LOH
}

// ✅ Zapište přímo do streamu/pipe – žádná velká alokace
public async Task SerializujDoStreamAsync(List<Polozka> polozky, Stream vystup, CancellationToken ct)
{
    await JsonSerializer.SerializeAsync(vystup, polozky, cancellationToken: ct);
}
```

## 3. Kdy se zaměřit na GC optimalizaci

- Aplikace má > 50 Gen 2 sbírek za minutu (měřte přes `GC.CollectionCount(2)` nebo Prometheus metriky).
- Latence P99 je výrazně vyšší než průměr – příznak GC pauz.
- Paměť roste pomalu ale trvale – příznak memory leaku (živé reference na zapomenuté objekty).
- Aplikace pracuje s velkými buffery nebo streamy dat (soubory, serializace, síťové packety).

## 4. Časté chyby

- ❌ **Statické kolekce, do kterých přidáváte bez čištění** – nikdy se neuvolní; klasický memory leak.
- ❌ **Event handlery bez odregistrace** – `source.Event += handler` udržuje zdroj živý, dokud je subscriber. Vždy odregistrujte nebo použijte `WeakReference`.
- ❌ **Zbytečné volání `GC.Collect()`** – narušuje generační model, způsobuje přesun objektů do Gen 2. Nikdy nevolat v produkci bez pečlivého odůvodnění.
- ❌ **Alokace pole > 85 KB jako krátkodobé objekty** – jdou na LOH a fragmentují ho. Použijte `ArrayPool<byte>` nebo `MemoryPool<T>`.
- ❌ **Pinning objektů přes `fixed` nebo `GCHandle`** – zablokuje kompakci GC. Pinujte co nejkratší dobu.

---

<details>
<summary>Deep dive: Server GC konfigurace, LOH kompakce a diagnostické nástroje</summary>

### Konfigurace GC v runtimeconfig.json

```json
{
  "configProperties": {
    "System.GC.Server": true,
    "System.GC.Concurrent": true,
    "System.GC.HeapHardLimit": 1073741824
  }
}
```

`HeapHardLimit` je klíčové v Kubernetes – bez něj GC neví o memory limitu kontejneru a může ho překročit, což vede k OOMKilled.

### Manuální kompakce LOH

```csharp
// Výchozí: LOH se nekompaktuje. Lze vynutit (drahé – použít jen mimořádně):
GCSettings.LargeObjectHeapCompactionMode = GCLargeObjectHeapCompactionMode.CompactOnce;
GC.Collect(); // Toto je výjimka z pravidla "nevolat GC.Collect"
```

### Diagnostické nástroje

| Nástroj | Použití |
|---------|---------|
| `dotnet-counters` | Live metriky GC (gen counts, heap size, alloc rate) |
| `dotnet-dump` | Heap dump pro analýzu živých objektů |
| `dotnet-trace` | ETW/EventPipe trace pro GC události |
| PerfView | Detailní GC analýza, flame grafy alokací |
| Visual Studio Diagnostic Tools | Integrovaný memory profiler pro lokální debugging |

```bash
# Live sledování GC metrik
dotnet-counters monitor --process-id <pid> System.Runtime

# Heap dump pro analýzu memory leaku
dotnet-dump collect --process-id <pid>
dotnet-dump analyze <dump-soubor>
> dumpheap -stat
```

### Generational Aware Analysis

Nástroj `dotnet-trace` zachytí GC události a ukáže, co způsobuje Gen 2 sbírky:

```bash
dotnet-trace collect --process-id <pid> --providers Microsoft-Windows-DotNETRuntime:0x1:4
```

Hledejte objekty, které zbytečně přežívají do Gen 2 – typicky cache bez expirace, statické slovníky, zapomenuté event handlery.

</details>

**Další krok:** [Span<T> a Memory<T> – zero-copy práce s pamětí](Span_Memory.md)
