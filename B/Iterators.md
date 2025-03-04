# Iterátory a `yield return`

Iterátory umožňují psát metody, které **vrací prvky postupně** – bez nutnosti alokovat celou kolekci předem. Klíčové slovo `yield return` pozastaví metodu a vrátí jeden prvek; při dalším volání pokračuje od místa pozastavení.

## 1. Koncept

Metoda s `yield return` vrací `IEnumerable<T>` nebo `IEnumerator<T>`. Kompilátor ji přeloží na stavový automat (state machine třídu). Tělo metody se nevykoná při volání – vykoná se **lazy**, až při iteraci přes `foreach`.

`yield break` ukončí iteraci předčasně (ekvivalent `return` v normální metodě).

## 2. Příklad

### Základní generátor

```csharp
// Metoda nevytvoří List – generuje prvky lazy
IEnumerable<int> Fibonacciho(int pocet)
{
    int a = 0, b = 1;
    for (int i = 0; i < pocet; i++)
    {
        yield return a;       // pozastaví, vrátí a
        (a, b) = (b, a + b);  // pokračuje zde při příštím MoveNext()
    }
}

foreach (int fib in Fibonacciho(8))
    Console.Write($"{fib} "); // 0 1 1 2 3 5 8 13
```

### Filtrace a transformace

```csharp
// Vlastní Where – lazy, nealokuje výslednou kolekci
IEnumerable<T> MujWhere<T>(IEnumerable<T> zdroj, Func<T, bool> podminka)
{
    foreach (T item in zdroj)
        if (podminka(item))
            yield return item;
}

// Stránkování bez ToList()
IEnumerable<T> Stranka<T>(IEnumerable<T> data, int stranka, int velikost)
{
    int preskocit = stranka * velikost;
    int vraceno   = 0;

    foreach (T item in data)
    {
        if (preskocit-- > 0) continue;
        if (vraceno++ >= velikost) yield break;
        yield return item;
    }
}
```

### yield break pro podmíněné ukončení

```csharp
IEnumerable<string> CtiRadkyDoKonce(StreamReader reader, string sentinel)
{
    string? radek;
    while ((radek = reader.ReadLine()) != null)
    {
        if (radek == sentinel)
            yield break;  // ukončení bez výjimky
        yield return radek;
    }
}
```

### Kdy lazy hodnocení ušetří práci

```csharp
// Bez iterátoru: prochází celou kolekci a alokuje List
List<Produkt> draheProdukty = produkty
    .Where(p => p.Cena > 1000)
    .ToList(); // alokace

// Lazy: zastaví se po nalezení prvního
Produkt? prvni = produkty
    .Where(p => p.Cena > 1000)
    .FirstOrDefault(); // prochází jen do prvního nalezeného
```

## 3. Kdy použít

- **Velké nebo nekonečné sekvence** – generování čísel, načítání řádků ze souboru, stránkování DB výsledků bez načtení všeho do paměti.
- **Vlastní LINQ operátory** – `Where`, `Select`, `Take` implementace pro vlastní zdroje dat.
- **Pipeline zpracování** – řetězení iterátorů kde každý krok transformuje prvky lazy.

**Nepoužívejte** pokud potřebujete náhodný přístup (indexování), opakovanou iteraci bez alokace, nebo výsledek sdílet s více konzumenty – tam je `List<T>` nebo `Array` vhodnější.

## 4. Časté chyby

- ❌ **Vícenásobná iterace** – `IEnumerable<T>` z iterátoru se vyhodnotí znovu při každém `foreach`. Pokud je generování drahé, zavolejte `.ToList()` a iterujte list.
- ❌ **Výjimky při lazy vyhodnocení** – výjimka vzniklá uvnitř iterátoru se projeví až při iteraci, ne při volání metody. Validaci parametrů oddělte do wrapper metody volané okamžitě.
- ❌ **`yield return` v `try/catch` s `catch`** – C# zakazuje `yield` v catch bloku (ale povoluje v `try` s `finally`). Zpracujte výjimku mimo `yield`.
- ❌ **Modifikace zdrojové kolekce během iterace** – `InvalidOperationException` stejně jako u `List<T>.GetEnumerator()`.

**Další krok:** [Async/await – základy](Async_await.md)
