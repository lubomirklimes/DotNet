# DotNet

## A. Úvod do .NET a jazyka c#

### **1\. Úvod do programování a C#**

-   [Uvod](1/Intro.md)
-   [Pojmy](1/Concepts.md)
-   [Podmínky a rozhodování](1/Conditions.md)
-   [Smyčky a opakování](1/Repeating.md)
-   [Funkce a metody](1/Methods.md)
-   [Pole a seznamy](1/Lists.md)
-   [Objektově orientované programování (OOP)](1/OOP.md)

## B. Poznámky k pokročilejším tématům .NETu:

### 1\. **Výkonové optimalizace v .NET Core**

-   [Techniky pro profilování aplikací .NET Core](2/Profilovani_aplikaci.md)
-   [Minimalizace doby startu aplikací a warm-up](2/Minimalizace_doby_startu_aplikace.md)
-   [JIT (Just-In-Time) vs AOT (Ahead-Of-Time) kompilace](2/JIT_AOT.md)
-   [Optimalizace práce s I/O (souborový systém, síťové operace)](2/Optimalizace_IO.md)
-   [Cache a vyrovnávací paměť (caching strategies)](2/Cache.md)
-   [Optimalizace správy paměti a garbage collection](2/Sprava_pameti.md)
-   [Použití Span<T>, Memory<T> a dalších moderních struktur pro výkon](2/Span_Memory.md)
-   [Optimalizace alokace paměti pomocí `ArrayPool<T>`](2/ArrayPool.md)
-   [Optimalizace stringů a práce s UTF-8 v .NET](2/Stringy_utf8.md)
-   [Výhody a úskalí `ref struct` a `readonly struct`](2/Struct.md)
-   [Minimalizace přepínání vláken a Thread Pool Tuning](2/Prepinani_vlaken.md)
      
### 2\. **Asynchronní programování a paralelní zpracování**

-   [Pokročilé techniky s `async` a `await`](2/Async_await.md)
-   [Správné použití `Task`, `ValueTask` a jejich dopady na výkon](2/Task_ValueTask.md)
-   [Paralelní zpracování s pomocí `System.Threading.Channels`](2/Paralelni_zpracovani_Channels.md)
-   [Praktické příklady s .NET Parallel Library a PLINQ](2/TPL_PLINQ.md)
-   [Přehled paralelních kolekcí v .NET: BlockingCollection, ConcurrentQueue a další](2/Paralelni_kolekce.md)
-   `IAsyncEnumerable` a asynchronní streamování dat
    -   Představuje moderní techniku pro **asynchronní iteraci**, což je užitečné při práci s datovými proudy, kde jednotlivé prvky dorazí postupně a nelze je načíst najednou.
    -   Příklady: zpracování velkých datových zdrojů (např. soubory, databázové dotazy) pomocí asynchronních iterátorů.
-   Pokročilé použití `TaskCompletionSource`
    -   Umožňuje explicitně řídit dokončení úloh `Task`. Je vhodné pro složitější asynchronní scénáře, například při stavbě vlastních knihoven nebo orchestraci úloh v rámci různých vláken.
-   Výhody a nevýhody `ConfigureAwait`
    -   Diskuze o tom, kdy a proč používat `ConfigureAwait(false)` pro zamezení přepínání kontextu mezi vlákny a jak to ovlivňuje výkon a deadlocky.
-   Orchestrace úloh: `Task.WhenAll`, `Task.WhenAny`, `Task.WaitAny`
    -   Detailní průzkum funkcí pro **správu více asynchronních úloh** najednou a jejich správné použití. Může zahrnovat optimalizace, jak efektivně spravovat více úloh (například kontrola, které úlohy dokončit jako první).
-   CancellationToken a řízení zrušení úloh**
    -   Jak správně implementovat asynchronní úlohy, které podporují možnost zrušení pomocí `CancellationToken`. Toto je důležité zejména pro dlouhotrvající nebo potenciálně nekonečné operace.
-   Asynchronní paralelní vzory s `Parallel.ForEachAsync`
    -   .NET 6 přidal nový způsob, jak efektivně zpracovávat položky kolekce asynchronně a paralelně. Tento vzor umožňuje rozložení asynchronních úkolů mezi více vláken a zároveň zůstává non-blocking.
-   Interakce mezi asynchronním a synchronním kódem
    -   Jak efektivně volat asynchronní kód z neasynchronního prostředí a naopak, včetně správného zvládání deadlocků a kontextového přepínání.
