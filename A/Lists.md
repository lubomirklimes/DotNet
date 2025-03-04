# Pole a seznamy

Pole a seznamy umožňují uložit **více hodnot** pod jedním názvem.

## 1. Pole (`array`) – pevná velikost

```csharp
int[] cisla = { 10, 20, 30, 40, 50 };

Console.WriteLine(cisla[0]);        // 10  (indexování od nuly)
Console.WriteLine(cisla.Length);    // 5
```

Procházení pole:

```csharp
for (int i = 0; i < cisla.Length; i++)
    Console.WriteLine(cisla[i]);

// nebo jednodušeji:
foreach (int cislo in cisla)
    Console.WriteLine(cislo);
```

## 2. Seznam (`List<T>`) – dynamická velikost

```csharp
List<string> jmena = new List<string>();

jmena.Add("Petr");
jmena.Add("Anna");
jmena.Add("Karel");
jmena.Remove("Anna");

foreach (string jmeno in jmena)
    Console.WriteLine(jmeno);
// Petr
// Karel
```

Důležité metody `List<T>`:

| Metoda | Co dělá |
|---|---|
| `Add(prvek)` | Přidá prvek na konec |
| `Remove(prvek)` | Odebere první výskyt |
| `Count` | Počet prvků |
| `Contains(prvek)` | Obsahuje prvek? |
| `Clear()` | Vymaže všechny prvky |

## 3. Pole nebo seznam?

| | Pole | Seznam |
|---|---|---|
| Velikost | pevná | dynamická |
| Kdy použít | známý počet prvků | přidáváme / odebíráme prvky |

---

<details>
<summary>Volitelné: Vícerozmerná pole</summary>

```csharp
int[,] matice = {
    { 1, 2, 3 },
    { 4, 5, 6 }
};
Console.WriteLine(matice[0, 2]); // 3
Console.WriteLine(matice[1, 0]); // 4
```

</details>

---

**Další krok:** [Objektově orientované programování (OOP)](OOP.md)
