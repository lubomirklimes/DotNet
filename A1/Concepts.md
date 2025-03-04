2. Základní pojmy v C#
======================

Abychom mohli psát programy v C#, musíme pochopit základní stavební prvky jazyka.

---

### **1. Proměnné a datové typy**  

**Proměnná** je místo v paměti, kde si můžeme uložit hodnotu. Každá proměnná má **datový typ**, který určuje, jaký druh hodnot může obsahovat.

#### **Nejčastější datové typy v C#:**  

| Datový typ | Popis | Příklad |

|------------|------------|------------|

| `int` | Celé číslo | `int vek = 12;` |

| `double` | Desetinné číslo | `double teplota = 36.5;` |

| `char` | Jeden znak | `char pismeno = 'A';` |

| `string` | Textový řetězec | `string jmeno = "Petr";` |

| `bool` | Pravda/nepravda | `bool jeStudent = true;` |

#### **Příklad: Deklarace proměnných a výpis na obrazovku**

```csharp

using System;

class Program

{

    static void Main()

    {

        int vek = 12;

        double teplota = 36.5;

        string jmeno = "Petr";

        bool jeStudent = true;

        Console.WriteLine("Jméno: " + jmeno);

        Console.WriteLine("Věk: " + vek);

        Console.WriteLine("Teplota: " + teplota);

        Console.WriteLine("Je student? " + jeStudent);

    }

}

```

📌 **Poznámka:** Používáme **`Console.WriteLine()`**, která vypisuje text na obrazovku.

---

### **2. Vstup a výstup**  

Programy často potřebují získat data od uživatele. K tomu slouží příkaz **`Console.ReadLine()`**.

#### **Příklad: Program, který se zeptá na jméno a věk**

```csharp

using System;

class Program

{

    static void Main()

    {

        Console.Write("Jak se jmenuješ? ");

        string jmeno = Console.ReadLine();

        Console.Write("Kolik je ti let? ");

        int vek = Convert.ToInt32(Console.ReadLine());

        Console.WriteLine("Ahoj, " + jmeno + "! Je ti " + vek + " let.");

    }

}

```

✅ **Co se děje?**  

- `Console.ReadLine();` -- získá vstup od uživatele jako text.  

- `Convert.ToInt32(...);` -- převede text na číslo.

---

### **3. Komentáře v kódu**  

Komentáře slouží k vysvětlení kódu. C# je ignoruje při spuštění.

| Typ komentáře | Použití | Příklad |

|--------------|---------|---------|

| **Jednořádkový** | Začíná `//` | `// Toto je komentář` |

| **Víceřádkový** | Začíná `/*` a končí `*/` | `/* Toto je víceřádkový komentář */` |

#### **Příklad komentářů v kódu**

```csharp

using System;

class Program

{

    static void Main()

    {

        // Vypíše uvítací zprávu

        Console.WriteLine("Vítej v programu!");

        /* Tento program se uživatele zeptá na jméno

           a poté ho pozdraví. */

        Console.Write("Jak se jmenuješ? ");

        string jmeno = Console.ReadLine();

        Console.WriteLine("Ahoj, " + jmeno + "!");

    }

}

```

✅ **Komentáře pomáhají lépe porozumět kódu.**

---

### **Shrnutí**  

- Proměnné ukládají data, např. `int`, `string`, `bool`.  

- `Console.ReadLine()` umožňuje uživateli zadat vstup.  

- `Console.WriteLine()` vypisuje text na obrazovku.  

- Komentáře pomáhají popsat kód a usnadnit jeho pochopení.

🔹 **Další krok:** Nyní se podíváme na podmínky a rozhodování v C#! 🚀