# Výjimky a jejich ošetření

Výjimka (`Exception`) je chyba, která nastane za běhu programu. Správné ošetření chrání program před pádem.

## Základní struktura: `try` / `catch` / `finally`

```csharp
try
{
    int cislo = int.Parse("ne-cislo"); // vyvolavá FormatException
}
catch (FormatException ex)
{
    Console.WriteLine($"Chyba formátu: {ex.Message}");
}
catch (Exception ex)
{
    Console.WriteLine($"Obecná chyba: {ex.Message}");
}
finally
{
    Console.WriteLine("Toto proběhne vždy."); // i po výjimce
}
```

- `catch` chytá výjimky od nejspecifičtější po nejobecnější.
- `finally` se provede vždy – typicky pro þklid (uzavření souborů, spojení).

## Běžné typy výjimek

| Výjimka | Kdy nastane |
|---|---|
| `ArgumentNullException` | Null předaný tam, kde se neočekává |
| `ArgumentOutOfRangeException` | Index mimo pole |
| `InvalidOperationException` | Operace v neplatném stavu |
| `IOException` | Chyba při práci se souborem |
| `HttpRequestException` | Chyba síťového volání |

## Vyházení výjimek

```csharp
void NastavVek(int vek)
{
    if (vek < 0)
        throw new ArgumentOutOfRangeException(nameof(vek), "Věk musí být kladný.");
    // ...
}
```

## Vlastní výjimky

```csharp
class NedostatekProstredkuException : Exception
{
    public decimal Castka { get; }

    public NedostatekProstredkuException(decimal castka)
        : base($"Nedöstatek prostředků: chybí {castka} Kč.")
    {
        Castka = castka;
    }
}
```

---

<details>
<summary>Volitelné: When filtry a `ExceptionDispatchInfo`</summary>

```csharp
try { /* ... */ }
catch (Exception ex) when (ex.Message.Contains("timeout"))
{
    Console.WriteLine("Timeout!");
}
```

`ExceptionDispatchInfo` umožňuje předat výjimku mezi vlákny bez ztráty stack trace.

</details>

---

**Další krok:** [Logování a diagnostika](Logging.md)
