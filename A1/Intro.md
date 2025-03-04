### **1. Úvod do programování a C#**

#### **Co je programování?**  

Programování je způsob, jak naučit počítač provádět úkoly. Pomocí programovacího jazyka mu dáváme jasné pokyny, co má dělat.

#### **Co je C# a proč ho používat?**  

C# (čti „C sharp") je moderní jazyk vytvořený společností Microsoft. Používá se pro vývoj aplikací, her (např. v Unity), webových stránek a dalších programů.

#### **Co je potřeba k programování v C#?**  

Než začneme psát kód, musíme si připravit nástroje:  

1\. **.NET SDK** -- obsahuje vše potřebné pro běh C# programů.  

2\. **Vývojové prostředí (IDE)** -- nejlepší volbou je **Visual Studio Code** nebo **Visual Studio**.

#### **Jak nainstalovat potřebné nástroje?**  

1\. Stáhni a nainstaluj **.NET SDK** z [dotnet.microsoft.com](https://dotnet.microsoft.com/).  

2\. Stáhni a nainstaluj **Visual Studio Code** z [code.visualstudio.com](https://code.visualstudio.com/) nebo **Visual Studio** z [visualstudio.microsoft.com](https://visualstudio.microsoft.com/).

#### **Náš první program: „Hello, World!"**  

Jakmile máme vše nainstalované, napíšeme první program.

1\. Otevři Visual Studio Code nebo Visual Studio.  

2\. Vytvoř nový soubor s koncovkou **.cs** (např. `HelloWorld.cs`).  

3\. Do souboru vlož tento kód:

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

4\. Ulož soubor a spusť program pomocí příkazu v terminálu:  

   ```

   dotnet run

   ```

#### **Co dělá tento kód?**  

- `using System;` -- umožňuje používat základní funkce C#.  

- `class Program` -- definuje třídu, která obsahuje náš kód.  

- `static void Main()` -- hlavní metoda, kterou C# spustí jako první.  

- `Console.WriteLine("Hello, World!");` -- vypíše text na obrazovku.

✅ **Hotovo! Právě jsi napsal svůj první program v C#!**