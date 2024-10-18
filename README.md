# DotNet
Poznámky k pokročilejším tématům .NETu:

## 1\. **Výkonové optimalizace v .NET Core**

-   [Techniky pro profilování aplikací .NET Core](Profilovani_aplikaci.md)
-   [Minimalizace doby startu aplikací a warm-up](Minimalizace_doby_startu_aplikace.md)
-   [JIT (Just-In-Time) vs AOT (Ahead-Of-Time) kompilace](JIT_AOT.md)
-   [Optimalizace práce s I/O (souborový systém, síťové operace)](Optimalizace_IO.md)
-   [Cache a vyrovnávací paměť (caching strategies)](Cache.md)
-   [Optimalizace správy paměti a garbage collection](Sprava_pameti.md)
-   [Použití Span<T>, Memory<T> a dalších moderních struktur pro výkon](Span_Memory.md)
-   [Optimalizace alokace paměti pomocí `ArrayPool<T>`](ArrayPool.md)
-   [Optimalizace stringů a práce s UTF-8 v .NET](Stringy_utf8.md)
-   [Výhody a úskalí `ref struct` a `readonly struct`](Struct.md)
-   [Minimalizace přepínání vláken a Thread Pool Tuning](Prepinani_vlaken.md)
      
## 2\. **Asynchronní programování a paralelní zpracování**

-   [Pokročilé techniky s `async` a `await`](Async_await.md)
-   [Správné použití `Task`, `ValueTask` a jejich dopady na výkon](Task_ValueTask.md)
-   [Paralelní zpracování s pomocí `System.Threading.Channels`](Paralelni_zpracovani_Channels.md)
-   [Praktické příklady s .NET Parallel Library a PLINQ](TPL_PLINQ.md)
-   [Přehled paralelních kolekcí v .NET: BlockingCollection, ConcurrentQueue a další](Paralelni_kolekce.md)
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
