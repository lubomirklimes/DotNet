Výhody a úskalí `ref struct` a `readonly struct` v .NET
=======================================================

Ve světě .NET programování hraje správná správa paměti a optimalizace výkonu klíčovou roli, zejména u aplikací s vysokými nároky na efektivitu a minimální latenci. Struktury (`struct`) představují jeden z nástrojů, jak dosáhnout vysokého výkonu díky práci s hodnotovými typy. Od zavedení `ref struct` a `readonly struct` v jazyce C# se tyto nástroje staly ještě silnějšími a umožnily vývojářům detailnější kontrolu nad tím, jak je paměť alokována a spravována. Tento článek se zaměří na podrobný rozbor těchto typů, jejich výhody, úskalí a ukázky toho, kdy a jak je efektivně využít pro optimalizaci výkonu a paměti.

* * * * *

### 1\. **Co je `ref struct`?**
-------------------------------

`ref struct` je speciální typ struktury, který je vždy alokován na zásobníku (`stack`) a nemůže být uložen v heap paměti. Na rozdíl od běžných struktur (`struct`), které mohou být součástí managed heapu (řízené paměti), je `ref struct` omezen tím, že musí být vždy předáván jako reference a jeho životnost je striktně omezena na rozsah metody nebo bloku, kde je používán.

#### Výhody `ref struct`:

-   **Nižší tlak na Garbage Collector (GC)**: Protože je `ref struct` alokován na zásobníku, nedochází k zátěži pro garbage collector, což může vést k lepšímu výkonu aplikace, zejména v reálném čase nebo u aplikací, které pracují s velkým množstvím krátkodobých objektů.
-   **Deterministická životnost**: `ref struct` je vždy uvolněn z paměti, jakmile opustí svou oblast platnosti (scope). To znamená, že vývojář má jistotu, že se nezdržuje v paměti déle, než je nutné.
-   **Rychlost přístupu**: Zásobník je rychlejší než heap pro alokaci i dealokaci paměti, což činí `ref struct` ideálním pro případy, kdy potřebujeme rychlý přístup k datům s nízkou režií.

#### Úskalí `ref struct`:

-   **Omezená životnost**: `ref struct` nemůže být uložen jako pole třídy nebo být součástí asynchronní metody (`async`). Tento typ nemůže být vracen jako výsledek z metody mimo jeho původní rozsah, což omezuje jeho použití v některých scénářích.
-   **Nekompatibilita s heap-based datovými strukturami**: `ref struct` nelze použít v kontextu, kde je potřeba jeho alokace na heapu (např. jako součástu třídy, `Task`, `IEnumerable`, apod.). To může představovat výzvu při návrhu API.

#### Typické použití `ref struct`:

`ref struct` je ideální pro práci s datovými strukturami, které jsou velmi citlivé na výkon a kde je nutné minimalizovat GC tlak. Jedním z příkladů je `Span<T>` a `ReadOnlySpan<T>`, které umožňují bezpečnou práci s paměťovými bloky bez nutnosti alokace na heapu.

```
public ref struct MyStruct
{
    private Span<byte> data;

    public MyStruct(Span<byte> data)
    {
        this.data = data;
    }
}
```


### 2\. **Co je `readonly struct`?**
------------------------------------

`readonly struct` je struktura, která označuje, že všechny její instance jsou neměnné (immutable). Toto klíčové slovo se používá k optimalizaci, kdy chcete zajistit, že žádné pole uvnitř struktury nemůže být modifikováno po jejím vytvoření. `readonly` struktury přinášejí jak výkonnostní, tak paměťové výhody.

#### Výhody `readonly struct`:

-   **Lepší optimalizace kompilátorem**: `readonly struct` umožňuje kompilátoru provádět optimalizace, protože ví, že instance struktury se nemění. Například může být zamezeno kopírování struktury při předávání hodnoty do metod, což snižuje režii.
-   **Bezpečnost kódu**: `readonly struct` zajišťuje, že všechny instance budou neměnné, což vede k jednoduššímu a méně chybovému kódu. To je velmi užitečné při práci s hodnotovými typy, které se předávají mezi metodami.
-   **Efektivnější při práci s `in` parametry**: `readonly struct` mohou být efektivněji předávány pomocí `in`parametrů, což znamená, že místo kopírování struktury se předává reference na hodnotu bez možnosti modifikace.

#### Úskalí `readonly struct`:

-   **Zvýšené požadavky na design**: Musíte pečlivě navrhnout strukturu tak, aby její pole byla skutečně neměnná. Pokud uvnitř `readonly struct` používáte mutable reference typy, nemusí být dosaženo plné neměnnosti.
-   **Nevhodné pro mutable scénáře**: Pokud potřebujete strukturu, která se v čase mění, `readonly struct` není vhodné řešení a může vést ke zbytečným komplikacím v kódu.

#### Typické použití `readonly struct`:

`readonly struct` je vhodné pro scénáře, kde chcete pracovat s neměnnými hodnotami, například u vektorových nebo matematických operací, kde je žádoucí pracovat s konstantními hodnotami bez obav o nechtěnou mutaci.

```
public readonly struct Vector2
{
    public readonly float X { get; }
    public readonly float Y { get; }

    public Vector2(float x, float y)
    {
        X = x;
        Y = y;
    }

    public float Length => MathF.Sqrt(X * X + Y * Y);
}
```

### 3\. **Kombinace `ref struct` a `readonly` -- Jak optimalizovat výkon?**
---------------------------------------------------------------------------

V některých scénářích je možné kombinovat výhody obou typů. Například při práci se strukturami, které reprezentují malé datové bloky (např. práci s pamětí nebo bufferem), může být efektivní použít `ref struct` pro minimalizaci alokací na heapu a `readonly` pro zajištění, že dané datové bloky zůstávají neměnné.

Při kombinaci těchto technik můžeme dosáhnout velmi efektivního využití paměti, kde je kód bezpečný a předvídatelný, a zároveň minimalizujeme tlak na GC a paměťovou náročnost.

* * * * *

### 4\. Závěr
-------------

**`ref struct` a `readonly struct`** v .NET poskytují vývojářům možnost optimalizovat výkon a paměťovou efektivitu aplikací. `ref struct` umožňuje kontrolu nad tím, kde a jak jsou struktury alokovány, čímž zajišťuje snížení tlaku na garbage collector a rychlejší přístup k datům. `readonly struct` naopak přináší bezpečnost a optimalizace díky neměnnosti, což vede k jednoduššímu a efektivnějšímu kódu.

Použití těchto struktur je vhodné v případě, kdy chcete **minimalizovat zátěž na GC**, **zajistit rychlý přístup k datům** a **mít plnou kontrolu nad tím, jak je kód spravován a optimalizován**. I když přicházejí s určitými omezeními, správné pochopení a využití `ref struct` a `readonly struct` může výrazně přispět k lepšímu výkonu a efektivitě vašich aplikací v .NET.