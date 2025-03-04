# Smyčky a opakování

Smyčky umožňují opakovat stejný kód vícekrát bez jeho opakování.

## 1. `for` – známý počet opakování

```csharp
for (int i = 1; i <= 5; i++)
{
    Console.WriteLine("Krok: " + i);
}
```

Výstup:
```
Krok: 1
Krok: 2
Krok: 3
Krok: 4
Krok: 5
```

## 2. `while` – opakuj, dokud platí podmínka

```csharp
int i = 1;
while (i <= 5)
{
    Console.WriteLine("Počítám: " + i);
    i++;
}
```

> **Pozor:** Pokud zapomeneš zvýšit `i`, smyčka poběží do nekonečna!

## 3. `do-while` – provéč alespoň jednou

```csharp
int cislo;
do
{
    Console.Write("Zadej číslo větší než 10: ");
    cislo = int.Parse(Console.ReadLine());
} while (cislo <= 10);

Console.WriteLine("Správně! Zadal jsi: " + cislo);
```

Podmínka se kontroluje až po prvním průchodu – kód proto proběhne vždy alespoň jednou.

## 4. `break` a `continue`

```csharp
for (int i = 1; i <= 10; i++)
{
    if (i == 3) continue;   // přeskoč 3
    if (i == 7) break;      // zakonči smyčku na 7
    Console.WriteLine(i);
}
// Vypíše: 1, 2, 4, 5, 6
```

## 5. `foreach` – procházení kolekce

```csharp
string[] barvy = { "modrá", "červena", "zelená" };
foreach (string barva in barvy)
{
    Console.WriteLine(barva);
}
```

`foreach` je ideální, když potřebujem projit všechny prvky kolekce.

---

<details>
<summary>Volitelné: Vnořené smyčky</summary>

Smyčku lze vložit do jiné smyčky – například pro výpis tabulky násobků:

```csharp
for (int i = 1; i <= 5; i++)
{
    for (int j = 1; j <= 5; j++)
    {
        Console.Write($"{i * j,4}");
    }
    Console.WriteLine();
}
```

Vnořené smyčky jsou užitečné, ale při špatném použití zpomalují program.

</details>

---

**Další krok:** [Funkce a metody](Methods.md)
