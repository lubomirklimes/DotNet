3\. Podmínky a rozhodování
=========================

Programy často potřebují **rozhodovat**, co mají udělat. V C# k tomu slouží **podmínky**.  

---

### **1. IF podmínky**  
Podmínky umožňují programu spustit určitý kód pouze tehdy, když je splněna nějaká **podmínka**.  
Používáme klíčové slovo **`if`**.

#### **Základní syntaxe:**
```csharp
if (podmínka)
{
    // Tento kód se provede, pokud je podmínka pravdivá (true)
}
```

#### **Příklad: Zjistíme, jestli je číslo kladné**
```csharp
using System;

class Program
{
    static void Main()
    {
        Console.Write("Zadej číslo: ");
        int cislo = Convert.ToInt32(Console.ReadLine());

        if (cislo > 0)
        {
            Console.WriteLine("Číslo je kladné.");
        }
    }
}
```
✅ Pokud uživatel zadá **5**, program vypíše „Číslo je kladné.“  
✅ Pokud zadá **-3**, program nic nevypíše (protože podmínka není splněna).  

---

### **2. ELSE – Co když podmínka není splněna?**  
Pokud chceme vykonat jiný kód, když podmínka **není splněna**, použijeme **`else`**.

#### **Příklad: Kladné nebo záporné číslo**
```csharp
using System;

class Program
{
    static void Main()
    {
        Console.Write("Zadej číslo: ");
        int cislo = Convert.ToInt32(Console.ReadLine());

        if (cislo > 0)
        {
            Console.WriteLine("Číslo je kladné.");
        }
        else
        {
            Console.WriteLine("Číslo je záporné nebo nula.");
        }
    }
}
```
✅ Pokud zadáme **5**, program vypíše „Číslo je kladné.“  
✅ Pokud zadáme **-3**, program vypíše „Číslo je záporné nebo nula.“  

---

### **3. ELSE IF – Více možností**  
Někdy máme více různých možností. Použijeme **`else if`**.

#### **Příklad: Hodnocení známky**
```csharp
using System;

class Program
{
    static void Main()
    {
        Console.Write("Zadej známku (1-5): ");
        int znamka = Convert.ToInt32(Console.ReadLine());

        if (znamka == 1)
        {
            Console.WriteLine("Výborně!");
        }
        else if (znamka == 2)
        {
            Console.WriteLine("Chvalitebně.");
        }
        else if (znamka == 3)
        {
            Console.WriteLine("Dobře.");
        }
        else if (znamka == 4)
        {
            Console.WriteLine("Dostatečně.");
        }
        else if (znamka == 5)
        {
            Console.WriteLine("Nedostatečně.");
        }
        else
        {
            Console.WriteLine("Neplatná známka!");
        }
    }
}
```
✅ Pokud zadáme **1**, program vypíše „Výborně!“.  
✅ Pokud zadáme **6**, program vypíše „Neplatná známka!“.  

---

### **4. Logické operátory**  
Můžeme kombinovat více podmínek pomocí logických operátorů:  

| Operátor | Význam | Příklad |
|----------|--------|---------|
| `&&` | A (obě podmínky musí být splněny) | `if (vek > 18 && vek < 30)` |
| `||` | Nebo (stačí, aby jedna podmínka byla splněna) | `if (vek < 10 || vek > 60)` |
| `!` | Negace (obrátí hodnotu) | `if (!jeStudent)` |

#### **Příklad: Vstup pouze pro teenagery (13–19 let)**
```csharp
using System;

class Program
{
    static void Main()
    {
        Console.Write("Zadej svůj věk: ");
        int vek = Convert.ToInt32(Console.ReadLine());

        if (vek >= 13 && vek <= 19)
        {
            Console.WriteLine("Jsi teenager!");
        }
        else
        {
            Console.WriteLine("Nejsi teenager.");
        }
    }
}
```
✅ Pokud zadáme **15**, program vypíše „Jsi teenager!“.  
✅ Pokud zadáme **25**, program vypíše „Nejsi teenager.“.  

---

### **5. SWITCH – Alternativa k if-else**  
Když máme **hodně možností**, můžeme použít **`switch`**.  

#### **Příklad: Převod čísla na den v týdnu**
```csharp
using System;

class Program
{
    static void Main()
    {
        Console.Write("Zadej číslo dne (1-7): ");
        int den = Convert.ToInt32(Console.ReadLine());

        switch (den)
        {
            case 1:
                Console.WriteLine("Pondělí");
                break;
            case 2:
                Console.WriteLine("Úterý");
                break;
            case 3:
                Console.WriteLine("Středa");
                break;
            case 4:
                Console.WriteLine("Čtvrtek");
                break;
            case 5:
                Console.WriteLine("Pátek");
                break;
            case 6:
                Console.WriteLine("Sobota");
                break;
            case 7:
                Console.WriteLine("Neděle");
                break;
            default:
                Console.WriteLine("Neplatné číslo!");
                break;
        }
    }
}
```
✅ Pokud zadáme **3**, program vypíše „Středa“.  
✅ Pokud zadáme **9**, program vypíše „Neplatné číslo!“.  

---

### **Shrnutí**  
- **`if`** spouští kód, pokud je podmínka pravdivá.  
- **`else`** spouští kód, pokud podmínka pravdivá není.  
- **`else if`** umožňuje více možností.  
- **Logické operátory (`&&`, `||`, `!`)** umožňují složitější podmínky.  
- **`switch`** je užitečný pro mnoho možností.  

🔹 **Další krok:** Naučíme se pracovat s cykly! 🚀