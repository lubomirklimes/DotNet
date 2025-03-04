# Struktury (`struct`)

`struct` je **hodnotový typ** – ukládá se přímo do paměti (na stack), ne na heap. Vhodný pro malé, jednoduché datové objekty.

## Základní použití

```csharp
struct Bod
{
    public int X;
    public int Y;

    public Bod(int x, int y) { X = x; Y = y; }
    public override string ToString() => $"({X}, {Y})";
}

Bod b = new Bod(10, 20);
Console.WriteLine(b); // (10, 20)
```

## Klíčový rozdíl: `struct` vs. `class`

```csharp
// struct - kopíruje hodnoty
Bod b1 = new Bod(5, 10);
Bod b2 = b1;
b2.X = 99;
Console.WriteLine(b1.X); // 5  (b1 zůstává nezměněn)

// class - předává odkaz
class BodClass { public int X; }
BodClass c1 = new BodClass { X = 5 };
BodClass c2 = c1;
c2.X = 99;
Console.WriteLine(c1.X); // 99  (c1 se změnilo)
```

| | `struct` | `class` |
|---|---|---|
| Paměť | stack | heap |
| Přiřazení | kopíruje | předává odkaz |
| Dědičnost | ne | ano |
| Vhodné pro | malá data (bod, barva) | složité objekty |

## `readonly struct` – nezměnitelná struktura

```csharp
readonly struct Barva
{
    public byte R { get; }
    public byte G { get; }
    public byte B { get; }

    public Barva(byte r, byte g, byte b) { R = r; G = g; B = b; }
}
```

`readonly struct` zabranní náhodné změně hodnot a může být efektivnější (compiler dostává více informací).

---

<details>
<summary>Volitelné: `ref struct` a výkonové struktury</summary>

`ref struct` může být uložen pouze na stacku – nelze ho umstit do kolekce ani na heap. Používá se pro vysoko-výkonné operace (viz `Span<T>`).

```csharp
ref struct TempBuffer
{
    public Span<int> Data;
}
```

</details>

---

**Další krok:** [Záznamy (record)](Record.md)
