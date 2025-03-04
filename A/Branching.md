# Podmínky a rozhodování

Programy často potřebují rozhodovat, co mají udělat. V C# k tomu slouží podmínky.

## 1. `if` / `else if` / `else`

```csharp
Console.Write("Zadej známku (1-5): ");
int znamka = int.Parse(Console.ReadLine());

if (znamka == 1)
    Console.WriteLine("Výborně!");
else if (znamka <= 3)
    Console.WriteLine("Obstojné.");
else if (znamka <= 5)
    Console.WriteLine("Slobě.");
else
    Console.WriteLine("Neplatná známka!");
```

## 2. Logické operátory

| Operátor | Význam | Příklad |
|---|---|---|
| `&&` | a zároveň | `vek >= 13 && vek <= 19` |
| `\|\|` | nebo | `vek < 6 \|\| vek > 65` |
| `!` | negace | `!jeStudent` |

```csharp
int vek = 16;
if (vek >= 13 && vek <= 19)
    Console.WriteLine("Jsi teenager!");
```

## 3. `switch` – více možností přehledně

```csharp
Console.Write("Zadej číslo dne (1-7): ");
int den = int.Parse(Console.ReadLine());

switch (den)
{
    case 1: Console.WriteLine("Pondělí"); break;
    case 2: Console.WriteLine("Úterý"); break;
    case 3: Console.WriteLine("Středa"); break;
    case 4: Console.WriteLine("Čtvrtek"); break;
    case 5: Console.WriteLine("Pátek"); break;
    case 6: Console.WriteLine("Sobota"); break;
    case 7: Console.WriteLine("Neděle"); break;
    default: Console.WriteLine("Neplatné číslo!"); break;
}
```

---

<details>
<summary>Volitelné: Ternární operátor a `switch` výraz</summary>

**Ternární operátor** – zkrácený zápis `if/else`:

```csharp
string vysledek = (cislo > 0) ? "kladné" : "záporné nebo nula";
```

**`switch` výraz** (moderní C#):

```csharp
string nazevDne = den switch
{
    1 => "Pondělí",
    2 => "Úterý",
    _ => "Jiný den"
};
```

</details>

---

**Další krok:** [Smyčky a opakování](Loops.md)
