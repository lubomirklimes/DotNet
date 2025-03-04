# DotNet

## A. Úvod do .NET a jazyka c#

### **1\. Úvod do programování a C#**

-   [Úvod do programování a C#](A1/Intro.md)
-   [Základní pojmy v C#](A1/Concepts.md)
-   [Podmínky a rozhodování](A1/Branching.md)
-   [Smyčky a opakování](A1/Loops.md)
-   [Funkce a metody](A1/Methods.md)
-   [Pole a seznamy](A1/Lists.md)
-   [Objektově orientované programování (OOP)](A1/OOP.md)

### **2. Základní API a knihovny .NET**

-   [Práce se soubory a adresáři](A2/File_IO.md)  
-   [Práce s datem a časem](A2/DateTime.md)  
-   [Základní kolekce a jejich použití](A2/Collections.md)  
-   [Řetězce a jejich zpracování](A2/Strings.md)  
-   [Výjimky a jejich ošetření](A2/Exceptions.md)  
-   [Logování a diagnostika](A2/Logging.md)  
-   [Základy práce se sítí](A2/Networking.md)  
-   [Serializace a deserializace](A2/Serialization.md)  
-   [Dependency Injection a práce se službami](A2/DependencyInjection.md)  

## B. Vývoj specifických typů aplikací

### **1. Vývoj desktopových aplikací**  

-   [Windows Forms (WinForms)](B1/WinForms.md)  
-   [WPF (Windows Presentation Foundation)](B1/WPF.md)  
-   [MAUI -- multiplatformní aplikace](B1/MAUI.md)

### **2. Vývoj webových aplikací**  

-   [ASP.NET Core MVC -- Klasický web](B2/ASP_NET_MVC.md)  
-   [Blazor -- Moderní webová aplikace v C#](B2/Blazor.md)  
-   [Minimal APIs -- Nejjednodušší způsob, jak vytvořit API](B2/MinimalAPI.md)

### **3. Vývoj mobilních aplikací**  

-   [Xamarin a MAUI -- Mobilní aplikace v .NET](B3/Xamarin.md)  
-   [Práce s GPS, senzory a notifikacemi](B3/Mobile_Features.md)

### **4. Cloud a mikroservisy**  

-   [Docker a Kubernetes v .NET](B4/Docker_Kubernetes.md)  
-   [Azure Functions a Serverless vývoj](B4/Azure_Functions.md)  
-   [Práce s databázemi v cloudu (SQL, NoSQL)](B4/Cloud_Databases.md)

### **5. Herní vývoj v .NET**  

-   [Unity a C# -- Herní engine](B5/Unity.md)  
-   [MonoGame -- Alternativa k Unity](B5/MonoGame.md)  
-   [Práce s fyzikou a kolizemi](B5/GamePhysics.md)

## C. Pokročilá témata .NET:

### 1\. **Výkonové optimalizace v .NET Core**

-   [Techniky pro profilování aplikací .NET Core](C1/Profilovani_aplikaci.md)
-   [Minimalizace doby startu aplikací a warm-up](C1/Minimalizace_doby_startu_aplikace.md)
-   [JIT (Just-In-Time) vs AOT (Ahead-Of-Time) kompilace](C1/JIT_AOT.md)
-   [Optimalizace práce s I/O (souborový systém, síťové operace)](C1/Optimalizace_IO.md)
-   [Cache a vyrovnávací paměť (caching strategies)](C1/Cache.md)
-   [Optimalizace správy paměti a garbage collection](C1/Sprava_pameti.md)
-   [Použití Span<T>, Memory<T> a dalších moderních struktur pro výkon](C1/Span_Memory.md)
-   [Optimalizace alokace paměti pomocí `ArrayPool<T>`](C1/ArrayPool.md)
-   [Optimalizace stringů a práce s UTF-8 v .NET](C1/Stringy_utf8.md)
-   [Výhody a úskalí `ref struct` a `readonly struct`](C1/Struct.md)
-   [Minimalizace přepínání vláken a Thread Pool Tuning](C1/Prepinani_vlaken.md)
      
### 2\. **Asynchronní programování a paralelní zpracování**

-   [Pokročilé techniky s `async` a `await`](C2/Async_await.md)
-   [Správné použití `Task`, `ValueTask` a jejich dopady na výkon](C2/Task_ValueTask.md)
-   [Paralelní zpracování s pomocí `System.Threading.Channels`](C2/Paralelni_zpracovani_Channels.md)
-   [Praktické příklady s .NET Parallel Library a PLINQ](C2/TPL_PLINQ.md)
-   [Přehled paralelních kolekcí v .NET: BlockingCollection, ConcurrentQueue a další](C2/Paralelni_kolekce.md)
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
