# DotNet
Poznámky k pokročilejším tématům .NETu:

## 1\. **Výkonové optimalizace v .NET Core**

-   [Techniky pro profilování aplikací .NET Core](Profilovani_aplikaci.md)
-   [Optimalizace správy paměti a garbage collection](Sprava_pameti.md)
-   [Minimalizace doby startu aplikací a warm-up](Minimalizace_doby_startu_aplikace.md)
-   [Použití Span<T>, Memory<T> a dalších moderních struktur pro výkon](Span_Memory.md)

-   JIT (Just-In-Time) vs AOT (Ahead-Of-Time) kompilace
    -   Rozbor výhod a nevýhod JIT a AOT kompilace v .NET. Jak tyto kompilace ovlivňují výkon běhu aplikace, startup time, a memory footprint.
    -   Možné pokrytí technologie **NativeAOT** (Ahead-of-Time Compilation) dostupné v .NET 7 a pozdějších verzích.
-   Výhody a úskalí `ref struct` a `readonly struct`
    -   Podrobný rozbor optimalizací paměti a výkonnosti pomocí struktur `ref` a `readonly`. Vhodné pro případy, kdy je třeba snížit tlak na garbage collection a zároveň zachovat vysoký výkon.
-   Optimalizace stringů a práce s UTF-8 v .NET
    -   Efektivní práce s textovými řetězci a `Span<char>`. Můžeš také zahrnout optimalizace pomocí UTF-8 stringů (novinka v .NET 7) a jak to ovlivňuje výkon při práci s textovými daty.
-   Cache a vyrovnávací paměť (caching strategies)
    -   Podrobnosti o správné implementaci cachingu, včetně **MemoryCache**, **DistributedCache**, a jak správně pracovat s paměťově náročnými operacemi pomocí vyrovnávací paměti.
-   Optimalizace práce s I/O (souborový systém, síťové operace)
    -   Asynchronní I/O, bufferování a optimalizace vstupně-výstupních operací. Pokrytí témat, jako je **BufferedStream** nebo práce se souborovým systémem pomocí **FileStream** optimalizovaného v novějších verzích .NET.
-   Minimalizace přepínání vláken a Thread Pool Tuning
    -   Pokročilá práce s vláknovým fondem (`ThreadPool`), jak upravovat jeho chování a využít možnosti správy vlákna tak, aby se snížila režie spojená s přepínáním vláken.
-   Optimalizace alokace paměti pomocí `ArrayPool<T>`
    -   Jak používat `ArrayPool` pro znovupoužití pole a minimalizaci alokací, což je užitečné především v situacích, kde dochází k častým dočasným alokacím.
      
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
