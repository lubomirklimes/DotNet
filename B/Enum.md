# Výčtové typy (`enum`)

`enum` definuje **pojmenovanou sadu konstantních hodnot**. Používá se tam, kde máme pevně daný seznam možností.

## Základní použití

```csharp
enum Den { Pondeli, Utery, Streda, Ctvrtek, Patek, Sobota, Nedele }

Den dnes = Den.Patek;
Console.WriteLine(dnes);        // Patek
Console.WriteLine((int)dnes);  // 4  (indexování od 0)
```

Hodnoty začínají od 0, ale lze je nastavit různě:

```csharp
enum StavObjednavky
{
    Nova = 1,
    Odeslana = 2,
    Dokoncena = 3,
    Zrusena = 4
}
```

## Převody

```csharp
// enum -> int
int cislo = (int)Den.Streda;   // 2

// int -> enum
Den den = (Den)5;               // Sobota

// string -> enum (bezpečné)
if (Enum.TryParse("Patek", out Den d))
    Console.WriteLine(d);       // Patek
```

## Použití ve `switch`

```csharp
Den dnes = Den.Sobota;
switch (dnes)
{
    case Den.Sobota:
    case Den.Nedele:
        Console.WriteLine("Víkend!"); break;
    default:
        Console.WriteLine("Pracovní den."); break;
}
```

---

<details>
<summary>Volitelné: `[Flags]` – kombinace hodnot</summary>

Atribut `[Flags]` umožňuje spojit více hodnot dohromady (každá hodnota musí být mocnina 2):

```csharp
[Flags]
enum Prava
{
    Zadna    = 0,
    Cteni    = 1,
    Zapis    = 2,
    Spusteni = 4,
    Vse      = Cteni | Zapis | Spusteni
}

Prava uzivatel = Prava.Cteni | Prava.Zapis;
Console.WriteLine(uzivatel);                           // Cteni, Zapis
Console.WriteLine(uzivatel.HasFlag(Prava.Zapis));      // True
```

</details>

---

**Použij `enum`, když:**
- máš pevný seznam možností (dny, stavy, barvy),
- chceš čitelnější kód (`Den.Pondeli` místo `0`).

**Další krok:** [Struktury (struct)](Struct.md)
