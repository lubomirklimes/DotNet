### Zvýšení výkonu v .NET pomocí Span<T>, Memory<T> a dalších moderních struktur

Správa paměti hraje klíčovou roli v optimalizaci výkonu aplikací. V .NET jsou k dispozici moderní struktury jako `Span<T>`, `Memory<T>`, `ReadOnlyMemory<T>` a další, 
které umožňují efektivnější práci s pamětí bez zbytečných alokací a kopírování dat. 

#### 1\. **Problém s tradičními datovými typy a správa paměti v .NET**

V tradičních aplikacích .NET se často používají typy jako `Array`, `List<T>`, nebo `String`, které mají několik nevýhod:

-   **Překopírovávání dat:** Při práci s částmi polí nebo řetězců se často vytváří nové instance, což vede k nadměrné alokaci paměti a přetížení garbage collectoru (GC).
-   **Zbytečné alokace:** Při každém kopírování dat nebo při rozdělení velkých struktur do menších se vytvářejí nové objekty na haldě.
-   **Výkon GC:** Velké množství malých alokací vede k častějším běhům GC, což zpomaluje aplikaci.

#### 2\. **Span<T> - Moderní struktura pro práci s daty v paměti**

`Span<T>` je typ zavedený v .NET Core 2.1, který umožňuje pracovat s bloky paměti bez zbytečných kopírování a alokací. Umožňuje efektivně přistupovat k paměťovým segmentům jak v managed paměti (např. v poli), tak i v unmanaged paměti (např. při práci s nativními API).

##### a) **Jak funguje `Span<T>`?**

-   `Span<T>` je **typ na zásobníku** (stack-allocated) a je **ref struct**, což znamená, že nemůže být uložen na haldě ani předán mezi thready.
-   `Span<T>` může odkazovat na část pole, segment řetězce nebo na paměť, kterou jste sami alokovali, aniž by bylo nutné vytvářet nové kopie těchto dat.

##### b) **Příklad použití `Span<T>`:**

Představme si, že máme pole čísel a potřebujeme pracovat pouze s jeho částí:

```
int[] numbers = { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };
Span<int> span = new Span<int>(numbers, 2, 5); // Od indexu 2, délka 5
foreach (int number in span)
{
    Console.WriteLine(number); // Vypíše 3, 4, 5, 6, 7
}
```

Tento přístup je velmi efektivní, protože nedochází k žádnému kopírování dat a `Span<T>` přímo odkazuje na existující paměť.

##### c) **Výhody `Span<T>`:**

-   **Žádné kopírování dat:** Pracujete přímo s původními daty.
-   **Efektivní správa paměti:** `Span<T>` je na zásobníku, což znamená rychlejší přístup a menší zátěž pro GC.
-   **Bezpečnost:** `Span<T>` kontroluje hranice při přístupu k datům, což zabraňuje přístupu mimo rozsah.

#### 3\. **Memory<T> - Práce s pamětí na haldě**

Na rozdíl od `Span<T>` je `Memory<T>` alokován na haldě a není omezen na životnost zásobníku. Díky tomu může být předán mezi různými thready a může být uložen jako člen třídy.

##### a) **Použití `Memory<T>`:**

Pokud potřebujete pracovat s daty po delší dobu, nebo pokud potřebujete předat paměť mezi thready, použijte `Memory<T>`:

```
int[] numbers = { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };
Memory<int> memory = new Memory<int>(numbers, 2, 5); // Stejné jako u Span<T>
Span<int> span = memory.Span; // Přístup ke Span<T> pro operace s daty
```

##### b) **Výhody `Memory<T>`:**

-   **Flexibilita:** Na rozdíl od `Span<T>` může být `Memory<T>` předáván mezi různými komponentami a třídami, aniž by byl omezen na životnost zásobníku.
-   **Možnost asynchronního použití:** `Memory<T>` může být použit v asynchronních metodách, kde životnost proměnných přesahuje rámec jedné metody.

#### 4\. **ReadOnlySpan<T> a ReadOnlyMemory<T>**

Kromě základních struktur `Span<T>` a `Memory<T>` existují jejich varianty pro práci s **pouze pro čtení** daty, tedy `ReadOnlySpan<T>` a `ReadOnlyMemory<T>`. Tyto typy poskytují stejnou výkonnost a flexibilitu, ale zaručují, že nedojde k modifikaci dat.

##### a) **Příklad s `ReadOnlySpan<T>`:**

Při práci s řetězci je ideální používat `ReadOnlySpan<char>`, což umožňuje efektivní práci s podřetězci bez zbytečného vytváření nových objektů `string`.

```
string text = "Hello, World!";
ReadOnlySpan<char> span = text.AsSpan(0, 5); // Výběr prvních 5 znaků
Console.WriteLine(span.ToString()); // Vypíše "Hello"
```

Tímto způsobem se můžete vyhnout nadměrným alokacím paměti při manipulaci s řetězci.

#### 5\. **Stackalloc - Efektivní alokace na zásobníku**

Pro ještě větší kontrolu nad pamětí můžete v .NET použít klíčové slovo `stackalloc`, které alokuje paměť přímo na zásobníku. To je užitečné pro scénáře, kde potřebujete rychlou a krátkodobou alokaci bez nutnosti správy haldy.

##### a) **Příklad s `stackalloc`:**

```
Span<int> numbers = stackalloc int[5]; // Alokace na zásobníku
numbers[0] = 42;
numbers[1] = 27;
foreach (int number in numbers)
{
    Console.WriteLine(number); // Vypíše 42, 27, 0, 0, 0
}
```

Alokace na zásobníku je rychlá a paměť je uvolněna automaticky, jakmile metoda skončí, což je vhodné pro scénáře, kde jsou data potřebná jen krátkodobě.

#### 6\. **Pokročilé případy použití**

Moderní struktury jako `Span<T>`, `Memory<T>` a `stackalloc` najdou uplatnění v různých scénářích, kde je klíčová vysoká výkonnost a kontrola nad pamětí:

-   **Manipulace s velkými soubory:** Při práci s velkými datovými bloky (např. soubory nebo binárními daty) můžete využít `Span<T>` a `Memory<T>` pro čtení a zápis bez zbytečných alokací.

-   **Komprese dat:** Algoritmy, které vyžadují manipulaci s velkými datovými bloky, mohou efektivně využívat `Span<T>` pro přístup k segmentům paměti.

-   **Síťová komunikace:** Při zpracování paketů nebo binárních zpráv můžete využít `Span<T>` pro přístup k různým částem dat bez nutnosti kopírování paměti.

#### 7\. **Závěr**

`Span<T>`, `Memory<T>`, `stackalloc` a další moderní struktury přinášejí výrazné vylepšení v oblasti správy paměti v .NET. Tyto struktury umožňují efektivnější práci s daty, snižují zátěž pro garbage collector a zvyšují výkon aplikací.
