# CLR, sestavení a přehled runtime

.NET aplikace neběží přímo na CPU – běží na **Common Language Runtime (CLR)**, který zajišťuje správu paměti, bezpečnost typů, načítání kódu a překlad do nativních instrukcí.

## 1. Koncept

### Common Language Runtime (CLR)

CLR je virtuální stroj .NET. Poskytuje:
- **Garbage Collector (GC)** – automatická správa paměti
- **JIT kompilátor** – překlad IL → nativní kód při prvním volání metody
- **Type system** – Common Type System (CTS) sdílený C#, F#, VB.NET
- **Exception handling** – strukturované zachytávání výjimek
- **Security** – code access security, type safety

### IL a sestavení (Assembly)

Kompilátor C# neprodukuje nativní kód – produkuje **Intermediate Language (IL)** uložený v `.dll` nebo `.exe` souborech (sestavení / assembly).

```
C# zdrojový kód
    ↓ csc / Roslyn kompilátor
IL bytecode (.dll / .exe)
    ↓ JIT kompilátor (při prvním volání)
Nativní CPU instrukce (x64, ARM64...)
```

Sestavení obsahuje:
- IL kód metod
- **Metadata** – popisy typů, metod, atributů (základ pro Reflection)
- **Manifest** – název, verze, závislosti

### AssemblyLoadContext

.NET 5+ nahradil `AppDomain` pro izolaci sestavení pomocí `AssemblyLoadContext` – umožňuje načítat různé verze stejné knihovny do izolovaných kontextů (plugin systémy, hot reload).

### JIT vs AOT

| | JIT (výchozí) | AOT (Native AOT) |
|---|---|---|
| Kompilace | Za běhu, poprvé při volání | Předem, při publishu |
| Startup | Pomalejší (první volání) | Velmi rychlý |
| Optimalizace | Runtime profil-guided | Statická |
| Velikost | Menší distribuce | Větší nativní binary |

→ Viz [H/JIT_AOT.md](../H/JIT_AOT.md) pro detailní srovnání a konfiguraci.

### GC přehled

GC automaticky uvolňuje objekty, na které neexistuje žádná živá reference. Pracuje generačně (Gen 0 → Gen 1 → Gen 2). Velké objekty (> 85 KB) jdou na Large Object Heap (LOH).

→ Viz [H/Sprava_pameti.md](../H/Sprava_pameti.md) pro detailní GC optimalizaci.

## 2. Příklad

### Reflection – přístup k metadatům za běhu

```csharp
// Čtení metadat sestavení
var assembly = Assembly.GetExecutingAssembly();
Console.WriteLine($"Název: {assembly.GetName().Name}");
Console.WriteLine($"Verze: {assembly.GetName().Version}");

// Dynamické vytvoření instance přes Reflection
Type typ = Type.GetType("MojeAplikace.Sluzba")!;
object instance = Activator.CreateInstance(typ)!;
MethodInfo metoda = typ.GetMethod("SpustiAsync")!;
await (Task)metoda.Invoke(instance, null)!;
```

### AssemblyLoadContext pro pluginy

```csharp
// Načtení pluginu v izolovaném kontextu
var context = new AssemblyLoadContext("Plugin", isCollectible: true);
Assembly plugin = context.LoadFromAssemblyPath("/plugins/MujPlugin.dll");

Type pluginTyp = plugin.GetType("MujPlugin.EntryPoint")!;
// ... použití pluginu ...

context.Unload(); // uvolní sestavení z paměti
```

### Zjištění runtime informací

```csharp
Console.WriteLine(RuntimeInformation.FrameworkDescription); // .NET 9.0.x
Console.WriteLine(RuntimeInformation.OSDescription);         // Windows 11 / Linux ...
Console.WriteLine(RuntimeInformation.ProcessArchitecture);   // X64 / Arm64
Console.WriteLine(Environment.ProcessorCount);               // počet CPU jader
```

## 3. Kdy je znalost CLR důležitá

- **Diagnostika výkonu** – chápete, proč JIT zahřívání ovlivňuje startup, jak GC způsobuje pauzy.
- **Plugin systémy** – `AssemblyLoadContext` pro dynamické načítání a uvolňování kódu.
- **Interop s nativním kódem** – P/Invoke, COM interop pracují na hranici CLR/nativní svět.
- **Source generators a Roslyn** – pracují s kompilátorem a IL přímo.
- **Trimming a Native AOT** – znalost assembly metadat pomáhá pochopit, co trimmer odstraní.

## 4. Časté chyby

- ❌ **Reflection v hot path** – `Type.GetMethod()` a `Invoke()` jsou mnohonásobně pomalejší než přímé volání. Pro výkonnostní kód použijte source generators nebo delegáty.
- ❌ **Předpoklad, že AppDomain izoluje** – v .NET Core / .NET 5+ existuje jen jeden AppDomain; izolace se provádí přes `AssemblyLoadContext` nebo separátní proces.
- ❌ **Ignorování warmup** – JIT přeloží metodu při prvním volání; benchmarky musí zahrnovat warmup nebo použít BenchmarkDotNet, který warmup zajistí automaticky.

**Další krok:** [Práce se soubory a adresáři](../C/File_IO.md)
