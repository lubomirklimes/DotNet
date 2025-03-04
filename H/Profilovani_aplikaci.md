# Profilování aplikací v .NET

Profilování je disciplína, při které místo spekulování o tom, co je pomalé, to změříte. Bez měření optimalizujete naslepo – a velmi často optimalizujete něco, co není bottleneck. Pravidlo je jednoduché: nejdříve změřte, pak optimalizujte.

## 1. Koncept

Existují dvě základní kategorie profilování:

**Sampling (statistické):** Profiler v pravidelných intervalech zastaví všechna vlákna a zaznamená, kde se nacházejí (call stack). Nízká overhead, vhodné pro produkci nebo dlouhé testy. Výsledkem je "flame graph" nebo "hot path" – seznam metod seřazených podle spotřeby CPU.

**Instrumentation (přesné):** Profiler vloží měřicí kód do každého vstupu/výstupu metody. Přesné výsledky, ale overhead může výrazně zpomalit aplikaci – nepoužívejte v produkci.

Pro .NET jsou klíčové dva zdroje dat: **CPU sampling** (kde tráví čas vlákna) a **alokační profiling** (co generuje tlak na GC).

## 2. Příklad

### BenchmarkDotNet – mikrobenchmarky

Nikdy neměřte výkon ručně přes `Stopwatch` pro porovnání dvou implementací. BenchmarkDotNet správně zahřeje JIT, eliminuje šum a dá statisticky platné výsledky.

```csharp
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;

[MemoryDiagnoser]   // Zobrazí alokace a GC počty
[SimpleJob]
public class StringBenchmark
{
    private readonly string[] _slova = Enumerable.Range(0, 1000)
        .Select(i => $"slovo_{i}").ToArray();

    [Benchmark(Baseline = true)]
    public string SpojeniPlusOperator()
    {
        var vysledek = "";
        foreach (var s in _slova) vysledek += s;
        return vysledek;
    }

    [Benchmark]
    public string SpojeniStringBuilder()
    {
        var sb = new StringBuilder();
        foreach (var s in _slova) sb.Append(s);
        return sb.ToString();
    }

    [Benchmark]
    public string SpojeniStringJoin() => string.Join("", _slova);
}

// Program.cs
BenchmarkRunner.Run<StringBenchmark>();
```

Typický výstup ukazuje: čas, alokace, počet Gen 0/1/2 sbírek. Hledejte třídu, která alokuje nejméně a je nejrychlejší – to je správná volba.

### dotnet-counters – live metriky

```bash
# Nainstalujte globální nástroj
dotnet tool install --global dotnet-counters

# Sledujte živě – bez přerušení aplikace
dotnet-counters monitor --process-id <pid> System.Runtime

# Výstup:
# [System.Runtime]
#   CPU Usage (%)                         12
#   GC Heap Size (MB)                    145
#   Gen 0 GC Count (delta)                 3
#   Gen 2 GC Count (delta)                 0  ← chceme vidět 0 nebo velmi málo
#   Allocation Rate (B/sec)          2345678  ← vysoké číslo = hodně alokací
#   ThreadPool Queue Length                0
```

### dotnet-trace – zachycení profilu

```bash
# Zachytí 30 sekund CPU sampling profilu
dotnet-trace collect --process-id <pid> --duration 00:00:30

# Otevřete výsledný .nettrace soubor ve Visual Studio nebo SpeedScope
# speedscope.app – online flame graph viewer
```

### Lokální profiling ve Visual Studiu

Pro lokální development: **Debug → Performance Profiler → CPU Usage + .NET Object Allocation**. Spusťte aplikaci, proveďte scénář, zastavte – VS ukáže flame graph a žhavé metody.

## 3. Kdy použít

- **BenchmarkDotNet** – při rozhodování mezi dvěma implementacemi (string manipulation, serializace, LINQ vs. for loop).
- **dotnet-counters** – kontinuální sledování alokací a GC v testovacím nebo staging prostředí.
- **dotnet-trace + flame graph** – identifikace CPU bottlenecku v produkci nebo při load testu.
- **dotnet-dump** – analýza memory leaku; zachytí snapshot haldy a umožní zjistit, co drží živé reference.
- **Visual Studio Diagnostic Tools** – rychlé lokální ladění bez instalace dalších nástrojů.

## 4. Časté chyby

- ❌ **Měření výkonu přes `Stopwatch` bez warmup** – JIT zkompiluje kód teprve při prvním volání; první výsledek je vždy zkreslený. BenchmarkDotNet toto řeší automaticky.
- ❌ **Profilování v Debug konfiguraci** – JIT optimalizace jsou v Debug vypnuty; výsledky neodpovídají produkci. Vždy profilujte Release build.
- ❌ **Optimalizace před měřením** – intuice o bottlenecku bývá špatná. V 80 % případů je nejpomalejší místo jinde, než si myslíte.
- ❌ **Ignorování alokací** – CPU profil ukazuje čas, ale velká část problémů s výkonem je v GC tlaku způsobeném alokacemi. Vždy zapněte `[MemoryDiagnoser]`.
- ❌ **Jednorázové měření** – výsledky se liší run-to-run. BenchmarkDotNet dělá statistiku; ručně spusťte test alespoň 5× a berte průměr.

---

<details>
<summary>Deep dive: PerfView, ETW události a profilování v Kubernetes</summary>

### PerfView – nejvýkonnější bezplatný nástroj

PerfView od Microsoftu zachytí ETW (Event Tracing for Windows) události přímo z jádra OS. Umí ukázat GC pauzy, JIT kompilaci, thread contention, I/O a CPU sampling v jednom souboru.

```
PerfView.exe /GCCollectOnly /AcceptEULA collect        # Pouze GC události
PerfView.exe /ThreadTime collect                        # CPU + thread pauzy
```

Klíčové pohledy:
- **GC Stats** – délky pauz, Gen 2 frekvence, LOH fragmentace
- **CPU Stacks** – flame graph CPU spotřeby
- **Thread Time** – kde vlákna čekají (lock contention, I/O, GC)

### Profilování v Kubernetes

V kontejneru nemáte GUI. Použijte:

```bash
# Zachyťte trace z běžícího podu
kubectl exec <pod> -- dotnet-trace collect --process-id 1 --duration 00:00:30 -o /tmp/trace.nettrace

# Stáhněte na lokální počítač
kubectl cp <pod>:/tmp/trace.nettrace ./trace.nettrace

# Analyzujte lokálně ve Visual Studiu nebo SpeedScope
```

### Application Insights / OpenTelemetry

Pro produkční monitoring používejte Application Insights nebo OpenTelemetry s metrikami .NET runtime:

```csharp
builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics => metrics
        .AddRuntimeInstrumentation()  // GC, ThreadPool, JIT metriky
        .AddAspNetCoreInstrumentation()
        .AddPrometheusExporter());
```

Nastavte alerty na `dotnet.gc.collections` (Gen 2) a `dotnet.gc.heap.total_allocated` – budete vědět o problémech dříve, než je nahlásí uživatelé.

</details>

**Další krok:** [Benchmarking s BenchmarkDotNet](Benchmarking.md)
