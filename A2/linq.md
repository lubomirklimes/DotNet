6\. Výrazy `LINQ` a zpracování kolekcí
======================================

`LINQ` (Language Integrated Query) je **dotazovací jazyk** pro kolekce v C#. Umožňuje snadno **filtrovat, řadit a transformovat data** pomocí jednoduché syntaxe.

---

## **1. Co je `LINQ`?**

`LINQ` umožňuje **psát SQL-podobné dotazy** nad kolekcemi (`List<T>`, polem, databází, XML apod.).

### **Základní dotaz `LINQ`**

```csharp

using System;

using System.Linq;

class Program

{

    static void Main()

    {

        int[] cisla = { 1, 2, 3, 4, 5, 6 };

        var sudaCisla = from c in cisla

                        where c % 2 == 0

                        select c;

        foreach (var c in sudaCisla)

            Console.WriteLine(c);

    }

}

```

✅ **Dotaz `LINQ` vypadá jako SQL**  

✅ **Filtrujeme sudá čísla (`where c % 2 == 0`)**

📌 **Klíčové slovo `using System.Linq;` je nutné pro LINQ!**

---

## **2. `LINQ` na `List<T>` -- Metodová syntaxe**

Metodová syntaxe je **kratší a přehlednější**:

```csharp

using System;

using System.Linq;

using System.Collections.Generic;

class Program

{

    static void Main()

    {

        List<int> cisla = new List<int> { 1, 2, 3, 4, 5, 6 };

        var sudaCisla = cisla.Where(c => c % 2 == 0);

        foreach (var c in sudaCisla)

            Console.WriteLine(c);

    }

}

```

✅ **Použití metody `Where()` místo `where` v dotazové syntaxi**  

✅ **Lambda `c => c % 2 == 0` funguje jako filtr**

📌 **Metodová syntaxe je běžnější než SQL-styl LINQ!**

---

## **3. `Select()` -- Výběr dat z kolekce**

```csharp

var vysledky = cisla.Select(c => c * c);

```

✅ **Každé číslo se umocní**  

✅ **`Select()` umožňuje transformaci dat**

---

## **4. `OrderBy()` a `OrderByDescending()` -- Řazení dat**

```csharp

var serazene = cisla.OrderBy(c => c);

```

✅ **Seřazení vzestupně**

```csharp

var sestupne = cisla.OrderByDescending(c => c);

```

✅ **Seřazení sestupně**

---

## **5. `First()`, `Last()`, `Single()` a `ElementAt()`**

```csharp

var prvni = cisla.First();

var posledni = cisla.Last();

var treti = cisla.ElementAt(2); // Index 2 = třetí prvek

```

✅ **Rychlý přístup k prvkům**

📌 **`Single()` vrací **jediný prvek**, jinak vyvolá chybu.**

---

## **6. `GroupBy()` -- Seskupování dat**

```csharp

var skupiny = studenti.GroupBy(s => s.Trida);

```

✅ **Skupiny podle třídy**  

✅ **Každá skupina obsahuje studenty z jedné třídy**

---

## **7. `Join()` -- Spojení dvou kolekcí**

```csharp

var spojeni = objednavky.Join(

    zakaznici,

    o => o.ZakaznikId,

    z => z.Id,

    (o, z) => new { o.Produkt, z.Jmeno }

);

```

✅ **Spojení `objednavky` a `zakaznici` podle ID**

---

## **8. `Aggregate()` -- Spojení hodnot do jedné**

```csharp

var suma = cisla.Aggregate((a, b) => a + b);

```

✅ **Sečtení všech čísel v kolekci**

---

## **9. `Any()` a `All()` -- Podmínky na kolekci**

```csharp

if (cisla.Any(c => c > 10)) { Console.WriteLine("Obsahuje velká čísla"); }

if (cisla.All(c => c > 0)) { Console.WriteLine("Všechny jsou kladné"); }

```

✅ **`Any()` zjistí, zda **nějaký prvek** splňuje podmínku**  

✅ **`All()` zjistí, zda **všechny prvky** splňují podmínku**

---

## **10. Kdy použít `LINQ`?**

✅ **Když pracujeme s kolekcemi a filtry**  

✅ **Když chceme čitelný kód místo smyček**  

✅ **Když transformujeme data (např. `Select()`)**

❌ **Nepoužívej `LINQ`, pokud:**  

- **Potřebuješ maximální výkon** (LINQ je méně efektivní než ruční smyčky).  

- **Dotaz je příliš složitý** -- lepší použít databázové dotazy (`SQL`).

---

## **11. Shrnutí**

- **`Where()`** -- filtruje prvky.  

- **`Select()`** -- transformuje hodnoty.  

- **`OrderBy()` a `OrderByDescending()`** -- řazení.  

- **`First()`, `Last()`, `Single()`** -- přístup k prvkům.  

- **`GroupBy()`** -- seskupování.  

- **`Join()`** -- spojení kolekcí.  

- **`Any()` a `All()`** -- podmínky na kolekci.

---

🔹 **Další krok:** Naučíme se pracovat se soubory! 🚀