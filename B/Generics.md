# Generics – generické typy a metody

Generics umožňují psát typy a metody, které pracují s **libovolným typem** určeným až při použití. Eliminují duplikaci kódu a přetypování a zachovávají typovou bezpečnost.

## 1. Koncept

Generický parametr `T` je zástupce za konkrétní typ. Kompilátor při kompilaci vytvoří specializovanou verzi pro každý použitý typ – žádné boxování pro hodnotové typy, žádné přetypování pro referenční.

**Omezení (constraints)** upřesňují, co se smí s `T` dělat:

| Constraint | Popis |
|---|---|
| `where T : class` |
| `where T : struct` | T musí být hodnotový typ |
| `where T : new()` | T musí mít bezparametrový konstruktor |
| `where T : ISomeInterface` | T musí implementovat rozhraní |
| `where T : BaseClass` | T musí dědit od třídy |
| `where T : notnull` | T nesmí být nullable |

## 2. Příklad

### Generická metoda a třída

```csharp
// Generická metoda – funguje pro jakýkoli typ
T[] Opakuj<T>(T hodnota, int pocet)
{
    var pole = new T[pocet];
    Array.Fill(pole, hodnota);
    return pole;
}

int[]    cisla  = Opakuj(42, 3);     // [42, 42, 42]
string[] slova  = Opakuj("ahoj", 2); // ["ahoj", "ahoj"]

// Generická třída – typově bezpečný zásobník
public class Zasobnik<T>
{
    private readonly Stack<T> _stack = new();
    public void Push(T item) => _stack.Push(item);
    public T Pop()           => _stack.Pop();
    public int Count         => _stack.Count;
}
```

### Omezení – práce s rozhraním

```csharp
// Constraint zajistí, že T implementuje IComparable<T>
T Vetsi<T>(T a, T b) where T : IComparable<T>
    => a.CompareTo(b) >= 0 ? a : b;

int    vetsiInt  = Vetsi(3, 7);       // 7
string vetsiStr  = Vetsi("abc", "xyz"); // "xyz"
```

### Generické rozhraní a kovariance/kontravariance

```csharp
// Kovariance (out) – IEnumerable<Derived> je přiřaditelné do IEnumerable<Base>
IEnumerable<string> stringy = new List<string> { "a", "b" };
IEnumerable<object> objekty = stringy; // OK díky kovariance

// Kontravariance (in) – Action<Base> je přiřaditelné do Action<Derived>
Action<object> zpracujObj = o => Console.WriteLine(o);
Action<string> zpracujStr = zpracujObj; // OK
```

## 3. Kdy použít

- **Kolekce a kontejnery** – `List<T>`, `Dictionary<K,V>`, vlastní datové struktury.
- **Repository a service pattern** – `IRepository<T>` místo duplikovaných metod pro každý entitní typ.
- **Utility metody** – swap, clamp, parsování, mapování – vždy stejná logika, různé typy.
- **Constraint `where T : new()`** – factory metody, které potřebují vytvořit instanci T.

## 4. Časté chyby

- ❌ **Reflection místo generics** – `Type.GetType()` a přetypování přes `object` je pomalé a ztrácí typovou bezpečnost. Generics jsou správná volba.
- ❌ **Příliš volné constraints** – bez constraintů nelze volat metody na T; bez `IComparable<T>` nelze porovnávat. Přidejte jen nezbytné constraints.
- ❌ **`where T : class` v situaci, kdy stačí `where T : notnull`** – zbytečně vylučuje hodnotové typy.
- ❌ **Statický stav v generické třídě** – `Foo<int>.Counter` a `Foo<string>.Counter` jsou **různé** statické proměnné. Každá specializace má vlastní.

**Další krok:** [Nullable reference types](Nullable.md)
