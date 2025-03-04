# Výrazy LINQ a zpracování kolekcí

LINQ (Language Integrated Query) umožňuje **filtrovat, řadit a transformovat** kolekce pomocí čitelné syntaxe.

## 1. Základní metody

```csharp
List<int> cisla = new List<int> { 1, 2, 3, 4, 5, 6, 7, 8 };

// Filtrování
var suda = cisla.Where(c => c % 2 == 0);           // 2, 4, 6, 8

// Transformace
var druhe_mocniny = cisla.Select(c => c * c);       // 1, 4, 9, 16, ...

// Řazení
var serazena = cisla.OrderByDescending(c => c);     // 8, 7, 6, ...

// Agregace
int soucet = cisla.Sum();                           // 36
int max    = cisla.Max();                           // 8
double avg = cisla.Average();                       // 4.5
```

## 2. Řetězení metod

```csharp
var vysledek = cisla
    .Where(c => c > 3)
    .Select(c => c * 2)
    .OrderBy(c => c);
// 8, 10, 12, 14, 16
```

## 3. Práce s objekty

```csharp
record Student(string Jmeno, int Znamka);

List<Student> studenti = new()
{
    new("Petr", 2),
    new("Anna", 1),
    new("Karel", 3)
};

// Setvříční podle známky a výpis jména
var vyborni = studenti
    .Where(s => s.Znamka == 1)
    .Select(s => s.Jmeno);
// Anna
```

## 4. `First`, `Any`, `All`, `Count`

```csharp
var prvni = cisla.First(c => c > 5);    // 6
bool nejaka = cisla.Any(c => c > 10);  // False
bool vsechna = cisla.All(c => c > 0);  // True
int pocet = cisla.Count(c => c % 2 == 0); // 4
```

---

<details>
<summary>Volitelné: SQL-styl syntaxe a `GroupBy`</summary>

LINQ lze psát také SQL-podobně:

```csharp
var suda = from c in cisla
           where c % 2 == 0
           orderby c
           select c;
```

Seskupování:

```csharp
var skupiny = studenti.GroupBy(s => s.Znamka);
foreach (var skupina in skupiny)
{
    Console.WriteLine($"Známka {skupina.Key}:");
    foreach (var s in skupina)
        Console.WriteLine($"  {s.Jmeno}");
}
```

</details>

---

> **Pozor na výkon:** LINQ je pohodlný, ale pro výkónné smyčky nad miliony prvků může být pomalejší než ruční `for`.

**Další krok:** [Generics – generické typy a metody](Generics.md)
