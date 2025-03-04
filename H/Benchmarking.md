# BenchmarkDotNet

BenchmarkDotNet je de-facto standard pro přesné měření výkonu .NET kódu. Odstraňuje statistické šumy způsobené JIT kompilací, GC a dalšími runtime faktory a produkuje reprodukovatelné výsledky.

## 1. Koncept

Naivní měření výkonu pomocí `Stopwatch` je nepřesné — JIT kompilace, GC pauses a CPU cache efekty zkreslují výsledky. BenchmarkDotNet tyto problémy řeší:

- **Warmup iterace** — JIT zkompiluje kód před měřením
- **Statistická analýza** — průměr, medián, směrodatná odchylka, percentily
- **Diagnostika** — paměťové alokace (`[MemoryDiagnoser]`), hardware čítače (`[HardwareCounters]`)
- **Parametrizace** — `[Params]` spustí benchmark pro více vstupních hodnot
- **Porovnání** — více metod v jedné třídě běží ve stejném prostředí

Výsledky jsou exportovány jako tabulka v konzoli, Markdown, HTML nebo JSON.

## 2. Příklad

### Základní benchmark

```csharp
// dotnet add package BenchmarkDotNet
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;

[MemoryDiagnoser]
[SimpleJob(RuntimeMoniker.Net80)]
public class StringBenchmark
{
    private const int Pocet = 1000;

    [Benchmark(Baseline = true)]
    public string Concatenation()
    {
        var s = "";
        for (int i = 0; i < Pocet; i++) s += i;
        return s;
    }

    [Benchmark]
    public string StringBuilder()
    {
        var sb = new System.Text.StringBuilder();
        for (int i = 0; i < Pocet; i++) sb.Append(i);
        return sb.ToString();
    }

    [Benchmark]
    public string StringCreate()
        => string.Concat(Enumerable.Range(0, Pocet).Select(i => i.ToString()));
}

// Program.cs
BenchmarkRunner.Run<StringBenchmark>();
```

### Parametrizace s [Params]

```csharp
[MemoryDiagnoser]
public class SearchBenchmark
{
    [Params(100, 10_000, 1_000_000)]
    public int N;

    private int[] _data = [];

    [GlobalSetup]
    public void Setup() => _data = Enumerable.Range(0, N).ToArray();

    [Benchmark]
    public int LinearSearch() => Array.IndexOf(_data, N / 2);

    [Benchmark]
    public int BinarySearch() => Array.BinarySearch(_data, N / 2);
}
```

### Porovnání verzí (baseline)

```csharp
[MemoryDiagnoser]
public class JsonBenchmark
{
    private readonly Objednavka _obj = new(42, "Kniha", 299.0m);
    private readonly string _json = """{"Id":42,"Produkt":"Kniha","Castka":299.0}""";

    [Benchmark(Baseline = true)]
    public string Serialize_NewtonsoftJson()
        => Newtonsoft.Json.JsonConvert.SerializeObject(_obj);

    [Benchmark]
    public string Serialize_SystemTextJson()
        => System.Text.Json.JsonSerializer.Serialize(_obj);

    [Benchmark]
    public Objednavka Deserialize_SystemTextJson()
        => System.Text.Json.JsonSerializer.Deserialize<Objednavka>(_json)!;
}

record Objednavka(int Id, string Produkt, decimal Castka);
```

### Spuštění

```bash
# VŽDY v Release konfiguraci!
dotnet run -c Release
# Výsledky v ./BenchmarkDotNet.Artifacts/
```

## 3. Kdy použít

- **Před a po optimalizaci** — naměřte baseline, proveďte změnu, porovnejte výsledky.
- **Výběr algoritmu nebo knihovny** — porovnání serializérů, kolekcí, hash algoritmů.
- **Regresní testy výkonu** — integrujte do CI; `BenchmarkDotNet.Exporters` generuje JSON pro porovnání s předchozím runnem.
- **Validace profiler výstupů** — profiler ukáže kde, BenchmarkDotNet změří kolik.

**Nepoužívejte** v Debug konfiguraci — výsledky jsou bezcenné bez JIT optimalizací.

## 4. Časté chyby

- ❌ **Spuštění v Debug nebo přes IDE** — BenchmarkDotNet na to upozorní varováním; vždy `dotnet run -c Release` z příkazové řádky.
- ❌ **Dead code elimination** — JIT může odstranit kód bez vedlejších efektů; vraťte hodnotu z benchmarku nebo použijte `Consumer` / `Volatile.Read`.
- ❌ **Příliš krátká metoda bez smyčky** — sub-nanosecundové operace jsou pod rozlišením měření; přidejte smyčku a vydělte čas počtem iterací pomocí `[OperationsPerInvoke(N)]`.
- ❌ **Sdílený stav mezi iteracemi** — výsledky z předchozí iterace mohou ovlivnit cache; izolujte stav v `[IterationSetup]` pokud je to nutné.

<details>
<summary>Deep-dive: Jak BenchmarkDotNet funguje uvnitř</summary>

### Životní cyklus měření

BenchmarkDotNet generuje pro každý benchmark **novou konzolovou aplikaci** (separátní proces), která:
1. Spustí **Pilot** fázi — určí optimální počet invokací na iteraci
2. Spustí **Warmup** iterace — JIT zkompiluje a cache se zahřeje
3. Spustí **Actual** iterace — naměří výsledky
4. Odešle data zpět přes named pipe do hlavního procesu

### Statistika výsledků

Výsledky obsahují:
- **Mean** — aritmetický průměr (pozor na outliery)
- **Median** — robustnější vůči outlierům
- **StdDev** — směrodatná odchylka (vysoká hodnota = nestabilní benchmark)
- **Ratio** — poměr vůči baseline metodě
- **Gen0/1/2** — počet GC kolekcí na 1000 invokací (z `[MemoryDiagnoser]`)
- **Allocated** — alokovaná paměť na invokaci

### Hardware Counters

```csharp
[HardwareCounters(HardwareCounter.BranchMispredictions, HardwareCounter.CacheMisses)]
public class SortBenchmark
{
    // ...
}
```

Vyžaduje ETW (Windows) nebo perf (Linux) s dostatečnými oprávněními.

### Exportéry a CI integrace

```csharp
[MarkdownExporter, JsonExporter]
public class MujBenchmark { }
```

Pro CI porovnání výsledků použijte `bencher` nebo vlastní skript porovnávající JSON výstupy.

### Micro vs Macro benchmark

- **Micro** (nanosekunda–mikrosekunda): BenchmarkDotNet je ideální; měřte izolované algoritmy
- **Macro** (milisekunda+): zvažte load testing nástroje (NBomber, k6) pro realistické scénáře pod zátěží

</details>

**Další krok:** [Minimalizace doby startu aplikací a warm-up](Minimalizace_doby_startu_aplikace.md)
