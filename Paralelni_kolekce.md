Přehled paralelních kolekcí v .NET: BlockingCollection, ConcurrentQueue a další
===============================================================================

V prostředí vícevláknového programování je správné řízení sdílených dat zásadní pro zachování bezpečnosti a efektivity aplikace. Tradiční kolekce, jako jsou seznamy, slovníky nebo fronty, nejsou bezpečné pro paralelní použití, protože nejsou navrženy pro práci s více vlákny. Pro řešení těchto výzev nabízí .NET framework několik kolekcí, které jsou navrženy tak, aby byly **thread-safe** a zároveň co nejvíce eliminovaly blokace a režijní náklady spojené se synchronizací.

V tomto článku se zaměříme na několik klíčových kolekcí, které jsou navrženy pro použití v paralelních scénářích, jako jsou:

-   **BlockingCollection**
-   **ConcurrentQueue**
-   **ConcurrentStack**
-   **ConcurrentBag**
-   **ConcurrentDictionary**
-   **Immutable Collections**

Každou z nich představíme, ukážeme praktické příklady a vysvětlíme, kdy je vhodné je použít.

* * * * *

1\. BlockingCollection - řízení toku dat mezi vlákny
----------------------------------------------------

`BlockingCollection<T>` je vysoce užitečná struktura pro scénáře producent-konzument. Umožňuje bezpečně vkládat a odebírat položky mezi vlákny a navíc poskytuje možnost blokování, když je kolekce plná nebo prázdná, což umožňuje snadné řízení toku dat mezi producenty a konzumenty.

### Hlavní vlastnosti:

-   **Podpora omezené kapacity**: Můžete nastavit maximální počet položek, které kolekce může obsahovat. To zabraňuje přetížení paměti při vysokém počtu úloh.
-   **Blokování při plné/prázdné kolekci**: Umožňuje blokující přístup pomocí metod `Take()` a `Add()`, což zabraňuje aktivnímu čekání.
-   **Podpora paralelních scénářů**: Umožňuje použití v aplikacích, kde je nutné synchronizovat producenty a konzumenty bez složitého řízení vláken.

### Praktický příklad:

```
using System;
using System.Collections.Concurrent;
using System.Threading.Tasks;

public class Example
{
    public static void Main()
    {
        BlockingCollection<int> collection = new BlockingCollection<int>(boundedCapacity: 5);

        Task producer = Task.Run(() =>
        {
            for (int i = 0; i < 10; i++)
            {
                collection.Add(i);
                Console.WriteLine($"Vloženo: {i}");
            }
            collection.CompleteAdding();  // Oznámení, že už nebude více dat
        });

        Task consumer = Task.Run(() =>
        {
            while (!collection.IsCompleted)
            {
                int item;
                if (collection.TryTake(out item, TimeSpan.FromMilliseconds(100)))
                {
                    Console.WriteLine($"Zpracováno: {item}");
                }
            }
        });

        Task.WaitAll(producer, consumer);
    }
}
```

V tomto příkladu `BlockingCollection` zajišťuje, že producent nemůže vkládat více než 5 položek najednou, a konzument vždy počká, dokud není k dispozici další položka ke zpracování.

### Kdy použít:

-   Ideální pro scénáře **producent-konzument**, kde je potřeba kontrolovat tok dat mezi různými vlákny.
-   Když je potřeba pracovat s omezeným množstvím paměti a zabránit přetížení systému.

2\. ConcurrentQueue - Nejsnadnější cesta k paralelní frontě (FIFO)
------------------------------------------------------------------

`ConcurrentQueue<T>` je bezpečná fronta pro vícevláknové prostředí, která poskytuje **neblokující** přístup. Využívá vzor FIFO (First-In, First-Out), což znamená, že položky jsou zpracovávány ve stejném pořadí, v jakém byly přidány.

### Hlavní vlastnosti:

-   **Nejsou žádné blokace**: Ve srovnání s `BlockingCollection` nemá `ConcurrentQueue` omezenou kapacitu ani blokující metody. Operace jsou neblokující, což minimalizuje režijní náklady.
-   **Rychlá práce s daty**: Efektivní zpracování velkého množství položek ve vícevláknovém prostředí.
-   **FIFO pořadí**: Data jsou zpracovávána ve stejném pořadí, v jakém byla vložena.

### Praktický příklad:

```
ConcurrentQueue<int> queue = new ConcurrentQueue<int>();

// Producent
Task producer = Task.Run(() =>
{
    for (int i = 0; i < 10; i++)
    {
        queue.Enqueue(i);
        Console.WriteLine($"Enqueued: {i}");
    }
});

// Konzument
Task consumer = Task.Run(() =>
{
    while (!queue.IsEmpty)
    {
        if (queue.TryDequeue(out int item))
        {
            Console.WriteLine($"Dequeued: {item}");
        }
    }
});

Task.WaitAll(producer, consumer);
```

V tomto příkladu vidíme jednoduché použití `ConcurrentQueue`, kde producent přidává položky do fronty a konzument je postupně odebírá.

### Kdy použít:

-   Když je nutné zpracovávat data ve vzoru **FIFO** bez blokování vlákna.
-   Když je potřeba rychlý přístup k datům v paralelním prostředí, například při implementaci **asynchronních front**.

3\. ConcurrentStack - Paralelní zásobník (LIFO)
-----------------------------------------------

`ConcurrentStack<T>` je thread-safe verze klasického zásobníku, kde jsou položky zpracovávány ve vzoru **LIFO** (Last-In, First-Out). Každé vlákno může současně přidávat a odebírat položky bez potřeby zámků.

### Hlavní vlastnosti:

-   **Nejspolehlivější pro LIFO zpracování**: Hodí se pro scénáře, kde poslední přidaná položka musí být zpracována jako první.
-   **Bez blokování**: Stejně jako `ConcurrentQueue`, i `ConcurrentStack` je neblokující a efektivně využívá více vláken.

### Praktický příklad:

```
ConcurrentStack<int> stack = new ConcurrentStack<int>();

Task producer = Task.Run(() =>
{
    for (int i = 0; i < 10; i++)
    {
        stack.Push(i);
        Console.WriteLine($"Pushed: {i}");
    }
});

Task consumer = Task.Run(() =>
{
    while (!stack.IsEmpty)
    {
        if (stack.TryPop(out int item))
        {
            Console.WriteLine($"Popped: {item}");
        }
    }
});

Task.WaitAll(producer, consumer);
```

Tento příklad demonstruje použití `ConcurrentStack`, kde producent tlačí položky na vrchol zásobníku a konzument je postupně odebírá.

### Kdy použít:

-   V situacích, kde potřebujete **LIFO** zpracování dat, například při implementaci zásobníku úloh.

4\. ConcurrentDictionary - Bezpečné zpracování klíč-hodnota párů
----------------------------------------------------------------

`ConcurrentDictionary<TKey, TValue>` je ideální pro scénáře, kde více vláken musí současně číst a zapisovat do stejné kolekce klíč-hodnota párů. Nabízí atomické operace pro přidávání, aktualizaci a odebírání hodnot.

### Hlavní vlastnosti:

-   **Thread-safe operace**: Zajišťuje bezpečný přístup k datům bez nutnosti manuální synchronizace.
-   **Atomické operace**: Operace jako `AddOrUpdate` a `GetOrAdd` jsou atomické, což znamená, že se provádějí celé bez přerušení, což zabraňuje race conditions.

### Praktický příklad:

```
ConcurrentDictionary<string, int> dictionary = new ConcurrentDictionary<string, int>();

// Producent
Task.Run(() =>
{
    for (int i = 0; i < 10; i++)
    {
        dictionary.TryAdd($"key-{i}", i);
    }
});

// Konzument
Task.Run(() =>
{
    for (int i = 0; i < 10; i++)
    {
        dictionary.AddOrUpdate($"key-{i}", 1, (key, oldValue) => oldValue + 1);
    }
});
```

Zde vidíme, jak `ConcurrentDictionary` umožňuje bezpečné přidávání a aktualizaci hodnot ve vícevláknovém prostředí.

### Kdy použít:

-   Pro správu **klíč-hodnota párů** v paralelních úlohách, kde je třeba více operací, které mohou probíhat současně.

5\. ConcurrentBag - Kolekce pro volné paralelní přidávání a odebírání dat
-------------------------------------------------------------------------

`ConcurrentBag<T>` je optimalizovaná pro scénáře, kde vlákna provádějí operace ukládání a odebírání položek bez ohledu na pořadí. Jedná se o kolekci, která umožňuje rychlý a neblokující přístup k datům. Každé vlákno si může uchovat vlastní instanci položek, což minimalizuje potřebu synchronizace mezi vlákny.

### Hlavní vlastnosti:

-   **Vysoká propustnost**: Optimalizovaná pro situace, kdy se k datům přistupuje bez požadavků na konkrétní pořadí.
-   **Nízká režie synchronizace**: Vlákna si mohou uchovávat vlastní kopii kolekce, čímž se snižuje potřeba synchronizace mezi nimi.
-   **Nezaručuje pořadí**: Na rozdíl od `ConcurrentQueue` nebo `ConcurrentStack` zde není garantováno, že položky budou zpracovány v určitém pořadí.

### Praktický příklad:

```
ConcurrentBag<int> bag = new ConcurrentBag<int>();

Task producer = Task.Run(() =>
{
    for (int i = 0; i < 10; i++)
    {
        bag.Add(i);
        Console.WriteLine($"Přidáno: {i}");
    }
});

Task consumer = Task.Run(() =>
{
    while (!bag.IsEmpty)
    {
        if (bag.TryTake(out int item))
        {
            Console.WriteLine($"Zpracováno: {item}");
        }
    }
});

Task.WaitAll(producer, consumer);
```

Tento příklad ukazuje, jak `ConcurrentBag` umožňuje producentům přidávat položky a konzumentům je zpracovávat. Nezáleží přitom na pořadí, v jakém jsou položky přidány nebo odebrány.

### Kdy použít:

-   Když potřebujete vysokou propustnost a na pořadí přidání a odebrání položek nezáleží.
-   Když není třeba synchronizovat pořadí zpracování a hlavním cílem je minimalizovat režii synchronizace.

6\. Immutable Collections - Kolekce pro bezpečné čtení bez zámků
----------------------------------------------------------------

`Immutable Collections` jsou kolekce, které **nelze měnit po jejich vytvoření**. Místo toho jakákoli změna vytváří novou verzi kolekce. To znamená, že vlákna mohou bezpečně číst z kolekce bez obav o race conditions nebo potřebu synchronizace. Immutable kolekce se často používají v prostředích, kde je více čtenářů a velmi málo zápisů, což eliminuje nutnost zámků a synchronizačních primitiv.

### Hlavní vlastnosti:

-   **Bezpečné čtení bez synchronizace**: Jelikož kolekce nelze měnit, vlákna mohou současně číst z jedné instance kolekce bez potřeby zámků.
-   **Vysoká čitelnost, nízká modifikovatelnost**: Ideální pro scénáře s častým čtením a vzácnými zápisy.
-   **Změny vytváří nové instance**: Každá změna kolekce vrací novou verzi, což může mít vyšší paměťové nároky, ale přináší bezpečí v paralelních prostředích.

### Příklad použití `ImmutableList`:

```
using System.Collections.Immutable;

ImmutableList<int> immutableList = ImmutableList.Create(1, 2, 3, 4);

// Pokus o změnu vytvoří novou verzi seznamu
ImmutableList<int> newList = immutableList.Add(5);

Console.WriteLine(string.Join(", ", newList)); // Výstup: 1, 2, 3, 4, 5
```

V tomto příkladu vidíme, jak je možné vytvářet nové instance seznamů při každé změně, což eliminuje nutnost synchronizace mezi vlákny.

### Kdy použít:

-   V prostředích s více čtenáři a minimálními zápisy, například v **systémech s vysokou frekvencí čtení**.
-   Když chcete zajistit, aby změny neměly vliv na jiné vlákna, která aktuálně čtou kolekci.

7\. Porovnání kolekcí a jejich použití
--------------------------------------

### Shrnutí jednotlivých kolekcí:

| Kolekce | Typ | Pořadí zpracování | Blokování | Ideální pro |
| --- | --- | --- | --- | --- |
| **BlockingCollection** | FIFO | Řízené kapacitou | Blokující operace pro řízení toku dat | Producent-konzument |
| **ConcurrentQueue** | FIFO | Zachováno | Ne | Paralelní fronty |
| **ConcurrentStack** | LIFO | Obrácené | Ne | Paralelní zásobníky |
| **ConcurrentBag** | N/A | Nezaručeno | Ne | Vysoká propustnost bez zámků |
| **ConcurrentDictionary** | N/A | N/A | Ne | Paralelní zpracování klíč-hodnota |
| **Immutable Collections** | N/A | N/A | Ne, nelze měnit | Více čtenářů, minimum zápisů |

Každá z těchto kolekcí má své místo v paralelním programování. Výběr vhodné kolekce závisí na typu úloh, které řešíte, a na požadavcích na **zachování pořadí**, **propustnost**, a na tom, zda potřebujete **synchronizaci mezi vlákny**.

8\. Kdy použít jednotlivé struktury?
------------------------------------

### Základní pravidla pro výběr kolekcí:

-   **Kdy použít `BlockingCollection`**: Pokud máte scénář **producent-konzument** a potřebujete omezit počet zpracovávaných položek nebo řídit tok dat. BlockingCollection umožňuje efektivní řízení kapacity a poskytuje blokování, což je vhodné pro vyrovnávání přenosu dat mezi vlákny.

-   **Kdy použít `ConcurrentQueue` nebo `ConcurrentStack`**: Když potřebujete **neblokující zpracování** ve vzoru **FIFO** (fronta) nebo **LIFO** (zásobník). Tyto struktury jsou vhodné pro rychlé a efektivní zpracování dat bez potřeby blokace vláken.

-   **Kdy použít `ConcurrentBag`**: Pokud máte vysokou propustnost dat a nepotřebujete zachovat žádné konkrétní pořadí položek. ConcurrentBag minimalizuje synchronizaci a je vhodný pro zpracování velkého množství dat, které mohou přicházet z různých vláken.

-   **Kdy použít `ConcurrentDictionary`**: V situacích, kde více vláken současně pracuje s daty klíč-hodnota. Typickým scénářem je potřeba atomických operací při přidávání, aktualizaci nebo odebírání hodnot.

-   **Kdy použít `Immutable Collections`**: Pokud máte více čtenářů, kteří potřebují bezpečně přistupovat k datům, a zápisy jsou vzácné. Immutable kolekce eliminují potřebu synchronizace a poskytují thread-safe přístup bez zámků.

* * * * *

Závěr
-----

Práce s kolekcemi v paralelním prostředí je klíčovou součástí vývoje aplikací, které musí zpracovávat data ve více vláknech nebo procesech. .NET framework poskytuje bohatou sadu **thread-safe kolekcí** navržených tak, aby minimalizovaly potřebu manuální synchronizace a zároveň poskytovaly vysoký výkon.

Znalost správného použití těchto kolekcí, jako jsou `BlockingCollection`, `ConcurrentQueue`, `ConcurrentBag` a další, vám umožní psát efektivní a výkonné paralelní aplikace. Každá z těchto kolekcí má své specifické výhody a je důležité zvolit správnou strukturu podle konkrétního scénáře.
