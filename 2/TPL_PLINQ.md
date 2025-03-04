Praktické příklady s .NET Parallel Library a PLINQ
==================================================

Asynchronní a paralelní programování se v moderním vývoji aplikací stává stále důležitějším, zejména v kontextu výkonných systémů s vysokou propustností. .NET poskytuje bohaté nástroje pro práci s paralelizací, mezi které patří **.NET Parallel Library (TPL)** a **Parallel LINQ (PLINQ)**. Tyto knihovny zjednodušují práci s více vlákny a umožňují snadno využívat vícejádrové procesory k zrychlení výpočtů.

* * * * *

### 1\. **Úvod do .NET Parallel Library (TPL)**
-----------------------------------------------

.NET Task Parallel Library (TPL) poskytuje API pro vytváření a řízení paralelních operací. Klíčovými třídami jsou zde `Task` a `Parallel`, které umožňují snadno rozdělit práci mezi více vlákna.

#### a) **Task Parallelism**

TPL umožňuje rozdělit úlohy na více nezávislých částí a spustit je současně. Klíčová třída je zde `Task`, která představuje asynchronní úlohu, která může být spuštěna paralelně.

**Příklad paralelního zpracování pomocí `Task`:**

```
var tasks = new List<Task>();

for (int i = 0; i < 5; i++)
{
    int taskId = i;
    tasks.Add(Task.Run(() =>
    {
        Console.WriteLine($"Úloha {taskId} začíná.");
        Task.Delay(1000).Wait();  // Simulace zpracování
        Console.WriteLine($"Úloha {taskId} končí.");
    }));
}

Task.WaitAll(tasks.ToArray());  // Čekání na dokončení všech úloh
```

V tomto příkladu vytváříme pět úloh, které se spustí paralelně. Každá úloha simuluje práci pomocí `Task.Delay` a vypíše informace o svém začátku a konci.

#### b) **Parallel Programming**

`Parallel` je dalším nástrojem TPL, který umožňuje paralelizaci běžných operací, jako jsou smyčky nebo akce na kolekcích. Metody `Parallel.For` a `Parallel.ForEach` poskytují jednoduchý způsob, jak paralelizovat iterace.

**Příklad použití `Parallel.For`:**

```
Parallel.For(0, 5, i =>
{
    Console.WriteLine($"Zpracovávám položku {i}");
    Task.Delay(1000).Wait();  // Simulace zpracování
});
```

Tento příklad ukazuje, jak lze pomocí `Parallel.For` paralelizovat jednoduchou smyčku. Každá iterace je vykonávána současně, což může výrazně urychlit zpracování v porovnání se sekvenčním provedením.

#### c) **Parallel.Invoke**

`Parallel.Invoke` umožňuje spustit více nezávislých akcí současně.

**Příklad:**

```
Parallel.Invoke(
    () => { Console.WriteLine("První úloha"); Task.Delay(1000).Wait(); },
    () => { Console.WriteLine("Druhá úloha"); Task.Delay(1000).Wait(); },
    () => { Console.WriteLine("Třetí úloha"); Task.Delay(1000).Wait(); }
);
```

Zde se tři akce spustí paralelně, což opět šetří čas ve srovnání se sekvenčním provedením.

### 2\. **Úvod do Parallel LINQ (PLINQ)**
-----------------------------------------

PLINQ je rozšířením LINQ, které umožňuje paralelizovat zpracování dotazů. Pokud pracujete s datovými kolekcemi, můžete snadno přidat paralelizaci pomocí metody `AsParallel()`, což vede k rychlejšímu zpracování, zejména u výpočetně náročných operací.

#### a) **Základní použití PLINQ**

PLINQ automaticky rozděluje dotazy na více vláken, čímž se dosahuje vyšší propustnosti. Například při práci s rozsáhlými datovými sadami může PLINQ zkrátit dobu zpracování.

**Příklad použití PLINQ:**

```
var numbers = Enumerable.Range(1, 20);

var results = numbers
    .AsParallel()
    .Where(n => n % 2 == 0)
    .Select(n => n * n);

foreach (var result in results)
{
    Console.WriteLine(result);
}
```

Tento příklad ukazuje, jak jednoduše lze pomocí `AsParallel()` paralelizovat LINQ dotazy. PLINQ automaticky rozdělí zpracování mezi dostupná jádra CPU.

#### b) **Řízení paralelizace s PLINQ**

Pomocí PLINQ lze kontrolovat, kolik vláken bude použito k paralelnímu zpracování. Metoda `WithDegreeOfParallelism` umožňuje omezit počet paralelních vláken.

**Příklad s omezením počtu vláken:**

```
var numbers = Enumerable.Range(1, 20);

var results = numbers
    .AsParallel()
    .WithDegreeOfParallelism(2)  // Omezí paralelizaci na 2 vlákna
    .Where(n => n % 2 == 0)
    .Select(n => n * n);

foreach (var result in results)
{
    Console.WriteLine(result);
}
```

Tímto způsobem můžeme omezit vytížení systému a vyhnout se přetížení CPU při paralelním zpracování rozsáhlých dat.

#### c) **Správné použití PLINQ**

PLINQ je skvělé pro datově intenzivní úlohy, ale nemusí být vhodné pro každou situaci. Paralelizace přináší režijní náklady (například na rozdělení práce a synchronizaci), což může být kontraproduktivní u menších datových sad nebo u úloh, které nelze efektivně rozdělit.

### 3\. **Kombinace TPL a PLINQ**
---------------------------------

TPL a PLINQ lze často používat společně, přičemž TPL poskytuje nízkoúrovňové řízení paralelizace, zatímco PLINQ usnadňuje práci s kolekcemi a datovými proudy.

#### Příklad: Kombinace TPL a PLINQ pro paralelní zpracování

Představme si scénář, kdy potřebujeme načíst a zpracovat velkou kolekci dat, například z databáze nebo API. Můžeme kombinovat TPL pro paralelní načítání a PLINQ pro paralelní zpracování dat.

**Příklad:**

```
var data = new List<int>();

// Paralelní načítání dat (např. z API nebo databáze)
Parallel.For(0, 10, i =>
{
    var partialData = GetDataFromSource(i);  // Simulace načítání dat
    lock (data)
    {
        data.AddRange(partialData);  // Synchronizovaný přístup k seznamu
    }
});

// Paralelní zpracování dat pomocí PLINQ
var processedData = data
    .AsParallel()
    .Select(n => ProcessData(n))  // Paralelní zpracování
    .ToList();
```

V tomto scénáři nejprve načteme data z několika zdrojů paralelně pomocí TPL a následně data zpracujeme pomocí PLINQ. Kombinace těchto dvou technik umožňuje maximalizovat výkon celého systému.

### 4\. **Výhody a omezení**
----------------------------

#### Výhody:

-   **Jednoduchost použití:** Obě knihovny nabízejí intuitivní API, které vývojářům umožňuje snadno paralelizovat úlohy bez nutnosti nízkoúrovňové správy vláken.
-   **Optimalizace výkonu:** Využití vícejádrových procesorů umožňuje rychlejší zpracování výpočetně náročných úloh.
-   **Škálovatelnost:** Paralelizace umožňuje aplikacím škálovat výkon s rostoucím počtem jader procesoru.

#### Omezení:

-   **Režijní náklady:** Paralelizace má určité režijní náklady, jako je rozdělení práce a synchronizace mezi vlákny. U malých úloh nebo dat může být paralelizace kontraproduktivní.
-   **Správa zdrojů:** Nesprávné používání paralelizace může vést k přetížení CPU a neefektivnímu využití systémových prostředků. Je důležité omezovat počet paralelních vláken a dbát na správnou synchronizaci sdílených zdrojů.

* * * * *

### 5\. **Závěr**
-----------------

.NET Task Parallel Library (TPL) a Parallel LINQ (PLINQ) jsou výkonné nástroje pro paralelizaci aplikací. TPL poskytuje nízkoúrovňové API pro řízení paralelních úloh, zatímco PLINQ zjednodušuje paralelní zpracování kolekcí dat. Obě knihovny lze snadno kombinovat a využívat k dosažení vyššího výkonu, zejména u výpočetně náročných úloh.

Při použití paralelizace je však důležité dbát na správné řízení zdrojů a zohlednit, zda je paralelizace ve vašem scénáři skutečně výhodná. Zvažte režijní náklady a potenciální přínos paralelizace podle charakteru vašich úloh a dat.
