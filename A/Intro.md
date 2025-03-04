# Úvod do programování a C#

## Co je programování?

Programování je způsob, jak dávat počítači pokyny. Pomocí programovacího jazyka říkáme počítači přesně, co má dělat – krok za krokem.

## Co je C#?

C# (čti „C sharp“) je moderní programovací jazyk od Microsoftu. Používá se pro:

- hry (engine Unity),
- desktopové a webové aplikace,
- mobilní aplikace a cloudové služby.

## Co budeš potřebovat?

| Nástroj | K čemu slouží | Odkaz |
|---|---|---|
| **.NET SDK** | Spouštění C# programů | [dotnet.microsoft.com](https://dotnet.microsoft.com/) |
| **Visual Studio** | Psaní a ladění kódu | [visualstudio.microsoft.com](https://visualstudio.microsoft.com/) |
| **VS Code** | Lehčí editor | [code.visualstudio.com](https://code.visualstudio.com/) |

## První program: Hello, World!

1. Otevři Visual Studio a vytvoř nový projekt **Console App**.
2. Do souboru `Program.cs` vlož tento kód:

```csharp
Console.WriteLine("Hello, World!");
```

3. Spušť program klávesou **F5** nebo příkazem v terminálu:

```
dotnet run
```

Výstup:
```
Hello, World!
```

---

<details>
<summary>Volitelné: Jak to vypadá v starším stylu zápisu?</summary>

Starší verze C# vyžadovaly více kódu – dnes ho .NET generuje automaticky:

```csharp
using System;

class Program
{
    static void Main()
    {
        Console.WriteLine("Hello, World!");
    }
}
```

- `using System;` – zpřístupní základní funkce.
- `class Program` – každý kód musí být ve třídě.
- `static void Main()` – vstupní bod programu.
- `Console.WriteLine(...)` – vypíše text na obrazovku.

</details>

---

**Další krok:** [Základní pojmy v C#](Concepts.md)
