6. Pole a seznamy
=================

Pole a seznamy umožňují ukládat **více hodnot** pod jedním názvem. Používají se pro práci s velkým množstvím dat.  

---

## **1. Pole (`array`)**  
Pole je **pevně daný** seznam hodnot stejného typu.  

### **Deklarace a inicializace pole**
```csharp
int[] cisla = { 1, 2, 3, 4, 5 };
```

📌 **Pole v C# mají pevnou velikost!**  
📌 **První prvek má index `0`** (číslování začíná od nuly).  

### **Příklad: Přístup k prvkům pole**
```csharp
using System;

class Program
{
    static void Main()
    {
        int[] cisla = { 10, 20, 30 };

        Console.WriteLine("První číslo: " + cisla[0]); 
        Console.WriteLine("Druhé číslo: " + cisla[1]); 
        Console.WriteLine("Třetí číslo: " + cisla[2]); 
    }
}
```
✅ **Výstup:**  
```
První číslo: 10  
Druhé číslo: 20  
Třetí číslo: 30  
```

---

### **2. Procházení pole pomocí smyčky**
Nejlepší způsob, jak projít všechny prvky pole, je smyčka.

#### **Příklad: Výpis prvků pole pomocí `for`**
```csharp
using System;

class Program
{
    static void Main()
    {
        int[] cisla = { 5, 10, 15, 20 };

        for (int i = 0; i < cisla.Length; i++)
        {
            Console.WriteLine("Prvek [" + i + "]: " + cisla[i]);
        }
    }
}
```
✅ **Výstup:**  
```
Prvek [0]: 5  
Prvek [1]: 10  
Prvek [2]: 15  
Prvek [3]: 20  
```

#### **Příklad: Výpis prvků pole pomocí `foreach`**
```csharp
using System;

class Program
{
    static void Main()
    {
        int[] cisla = { 3, 6, 9 };

        foreach (int cislo in cisla)
        {
            Console.WriteLine("Číslo: " + cislo);
        }
    }
}
```
✅ `foreach` se hodí, když nepotřebujeme indexy.

---

## **3. Seznamy (`List<T>`)**  
Seznamy (List) jsou **dynamická** pole – jejich velikost se může měnit.

### **Vytvoření seznamu**
```csharp
using System;
using System.Collections.Generic; // Nutné pro List<T>

class Program
{
    static void Main()
    {
        List<string> jmena = new List<string>();
    }
}
```
📌 `List<T>` se nachází v **`System.Collections.Generic`**.  

---

### **4. Přidávání a odebírání prvků v seznamu**  
Metody `Add()` a `Remove()` umožňují manipulaci s prvky.  

#### **Příklad: Přidání a odebrání hodnot**
```csharp
using System;
using System.Collections.Generic;

class Program
{
    static void Main()
    {
        List<string> jmena = new List<string>();

        jmena.Add("Petr");
        jmena.Add("Anna");
        jmena.Add("Karel");

        Console.WriteLine("Seznam jmen:");
        foreach (string jmeno in jmena)
        {
            Console.WriteLine(jmeno);
        }

        jmena.Remove("Anna"); // Odebrání Anny

        Console.WriteLine("Po odebrání Anny:");
        foreach (string jmeno in jmena)
        {
            Console.WriteLine(jmeno);
        }
    }
}
```
✅ **Výstup:**  
```
Seznam jmen:  
Petr  
Anna  
Karel  

Po odebrání Anny:  
Petr  
Karel  
```

---

### **5. Další užitečné metody pro seznamy**
| Metoda | Popis |
|--------|-------|
| `Add(prvek)` | Přidá prvek na konec seznamu |
| `Remove(prvek)` | Odebere první výskyt prvku |
| `Count` | Vrátí počet prvků v seznamu |
| `Contains(prvek)` | Vrátí `true`, pokud seznam obsahuje prvek |
| `Clear()` | Odstraní všechny prvky |

#### **Příklad: Použití `Count`, `Contains` a `Clear`**
```csharp
using System;
using System.Collections.Generic;

class Program
{
    static void Main()
    {
        List<int> cisla = new List<int> { 10, 20, 30, 40 };

        Console.WriteLine("Počet prvků: " + cisla.Count);
        Console.WriteLine("Obsahuje číslo 20? " + cisla.Contains(20));

        cisla.Clear(); // Vymazání všech prvků
        Console.WriteLine("Počet prvků po vymazání: " + cisla.Count);
    }
}
```
✅ **Výstup:**  
```
Počet prvků: 4  
Obsahuje číslo 20? True  
Počet prvků po vymazání: 0  
```

---

### **6. Seznamy vs. pole – Kdy co použít?**
| Typ | Kdy použít? |
|-----|------------|
| **Pole (`array`)** | Když máme **pevný počet prvků** a nepotřebujeme měnit velikost. |
| **Seznam (`List<T>`)** | Když potřebujeme **přidávat a mazat prvky** dynamicky. |

---

### **Shrnutí**  
✅ **Pole (`array`)** – pevná velikost, rychlý přístup k prvkům.  
✅ **Seznam (`List<T>`)** – dynamická velikost, obsahuje užitečné metody (`Add()`, `Remove()`).  
✅ **Použití `for` a `foreach`** pro procházení prvků.  
✅ **Metody seznamu (`Count`, `Contains`, `Clear`)** pro správu dat.  

🔹 **Další krok:** Naučíme se, jak funguje objektově orientované programování (OOP)! 🚀