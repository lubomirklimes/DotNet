# Základní kolekce a jejich použití

.NET nabízí řadu kolekcí pro různé potřeby. Správný výběr kolekce ovlivňuje čitelnost i výkon kódu.

## Přehled nejpoužívanějších kolekcí

| Kolekce | Kdy použít |
|---|---|
| `List<T>` | Obecný seznam, časté přidávání na konec |
| `Dictionary<K,V>` | Vyhledávání podle klíče |
| `HashSet<T>` | Unikátní prvky, rychlé hledání |
| `Queue<T>` | Fronta – FIFO (první dovnitř, první ven) |
| `Stack<T>` | Zásobník – LIFO (poslední dovnitř, první ven) |

## `Dictionary<K, V>`

```csharp
Dictionary<string, int> body = new()
{
    { "Petr", 85 },
    { "Anna", 92 }
};

body["Karel"] = 78; // přidání

Console.WriteLine(body["Anna"]); // 92

// Bezpečné čtení
if (body.TryGetValue("Petr", out int bodPetr))
    Console.WriteLine(bodPetr);

foreach (var (jmeno, hodnota) in body)
    Console.WriteLine($"{jmeno}: {hodnota}");
```

## `HashSet<T>`

```csharp
HashSet<string> zeme = new() { "CZ", "SK", "DE" };
zeme.Add("CZ");  // nemá efekt (duŤlikum ignorován)

Console.WriteLine(zeme.Count);           // 3
Console.WriteLine(zeme.Contains("SK")); // True
```

## `Queue<T>` a `Stack<T>`

```csharp
Queue<string> fronta = new();
fronta.Enqueue("první");
fronta.Enqueue("druhý");
Console.WriteLine(fronta.Dequeue()); // první

Stack<int> zasobnik = new();
zasobnik.Push(1);
zasobnik.Push(2);
Console.WriteLine(zasobnik.Pop()); // 2
```

---

<details>
<summary>Volitelné: `SortedDictionary`, `LinkedList` a nezmměnitelné kolekce</summary>

```csharp
// Automaticky seřazený slovník
SortedDictionary<string, int> sorted = new() { {"b", 2}, {"a", 1} };
// itěrace vrací: a=1, b=2

// Nezměnitelné kolekce (System.Collections.Immutable)
var list = ImmutableList.Create(1, 2, 3);
var list2 = list.Add(4); // vrátí novou kolekci
```

Pokročilejší kolekce viz sekci [C2 – Paralelní kolekce](../H/Paralelni_kolekce_concurrent.md).

</details>

---

**Další krok:** [Řetězce a jejich zpracování](Strings.md)
