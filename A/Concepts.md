# Základní pojmy v C#

## 1. Proměnné a datové typy

**Proměnná** je místo v paměti, kde si uložíme hodnotu. Každá proměnná má **datový typ**.

| Typ | Co ukládá | Příklad |
|---|---|---|
| `int` | celé číslo | `int vek = 15;` |
| `double` | desetinné číslo | `double teplota = 36.5;` |
| `string` | text | `string jmeno = "Petr";` |
| `bool` | pravda/nepravda | `bool jeStudent = true;` |
| `char` | jeden znak | `char pismeno = 'A';` |

```csharp
int vek = 15;
string jmeno = "Petr";
bool jeStudent = true;

Console.WriteLine($"Jméno: {jmeno}, věk: {vek}");
// Jméno: Petr, věk: 15
```

> **Tip:** Místo `"Jméno: " + jmeno` používej **interpolaci řetězců** `$"..."` – je přehlednější.

## 2. Vstup a výstup

```csharp
Console.Write("Jak se jmenuješ? ");
string jmeno = Console.ReadLine();

Console.Write("Kolik je ti let? ");
int vek = int.Parse(Console.ReadLine());

Console.WriteLine($"Ahoj, {jmeno}! Je ti {vek} let.");
```

- `Console.Write(...)` – vypíše text (bez odchodu na nový řádek).
- `Console.ReadLine()` – přečte vstup od uživatele jako text.
- `int.Parse(...)` – převede text na celé číslo.

## 3. Komentáře

Komentáře jsou ignorovány při spuštění – slouží jen pro vysvětlení kódu.

```csharp
// Jednorádkový komentář

/* Víceřádkový
   komentář */
```

---

<details>
<summary>Volitelné: Převody mezi typy</summary>

`int.Parse()` spadíme, pokud uživatel nezadá číslo. Bezpečnější varianta:

```csharp
if (int.TryParse(Console.ReadLine(), out int vek))
    Console.WriteLine($"Věk: {vek}");
else
    Console.WriteLine("To není číslo!");
```

Explicitní převod čísel: `double cislo = (double)5 / 2;` (výsledek: `2.5`)

</details>

---

**Další krok:** [Podmínky a rozhodování](Branching.md)
