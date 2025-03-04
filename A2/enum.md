1\. Výčtové typy (`enum`)
========================

`enum` (výčtový typ) umožňuje pojmenovat **konstantní sadu hodnot**. Používá se, když máme **pevně daný seznam možností** (např. dny v týdnu, stavy objednávky, barvy apod.).

---

## **1. Co je `enum`?**

`enum` je **hodnotový typ** (`value type`), kde každá hodnota má **číslo (integer)**.

### **Základní syntaxe `enum`**

```csharp

enum Den

{

    Pondeli,

    Utery,

    Streda,

    Ctvrtek,

    Patek,

    Sobota,

    Nedele

}

```

📌 **Poznámky:**  

- Hodnoty **automaticky začínají od 0** (`Pondeli = 0`, `Utery = 1`, ...).  

- `enum` je v **paměti uložen jako `int`**.

---

## **2. Jak používat `enum`?**

```csharp

using System;

enum Den

{

    Pondeli,

    Utery,

    Streda,

    Ctvrtek,

    Patek,

    Sobota,

    Nedele

}

class Program

{

    static void Main()

    {

        Den dnes = Den.Patek;

        Console.WriteLine("Dnes je: " + dnes); // Výstup: Dnes je: Patek

    }

}

```

✅ `enum` se chová jako běžná hodnota a lze ho snadno vypsat.

---

## **3. `enum` s vlastními čísly**

Můžeme ručně nastavit hodnoty:

```csharp

enum StavObjednavky

{

    Nova = 1,

    Odeslana = 2,

    Dokoncena = 3,

    Zrusena = 4

}

class Program

{

    static void Main()

    {

        StavObjednavky stav = StavObjednavky.Odeslana;

        Console.WriteLine((int)stav); // Výstup: 2

    }

}

```

📌 **Čísla nemusí být postupná, můžeme je měnit:**

```csharp

enum KodChyby

{

    OK = 0,

    Warning = 100,

    Error = 500

}

```

---

## **4. Převod `enum` na `int` a zpět**

```csharp

enum Barva

{

    Cervena = 1,

    Zelena = 2,

    Modra = 3

}

class Program

{

    static void Main()

    {

        Barva mojeBarva = Barva.Zelena;

        int cislo = (int)mojeBarva;

        Console.WriteLine(cislo); // Výstup: 2

        Barva zCisla = (Barva)3;

        Console.WriteLine(zCisla); // Výstup: Modra

    }

}

```

✅ `enum` můžeme snadno převádět na číslo a zpět.

---

## **5. Převod `string` ⇄ `enum`**

### **Převedení textu na `enum` (`Enum.Parse`)**

```csharp

enum Stav

{

    Aktivni,

    Neaktivni

}

class Program

{

    static void Main()

    {

        string vstup = "Neaktivni";

        Stav stav = (Stav)Enum.Parse(typeof(Stav), vstup);

        Console.WriteLine(stav); // Výstup: Neaktivni

    }

}

```

📌 **`Enum.Parse()` je case-sensitive** (rozlišuje velká/malá písmena).

### **Bezpečný převod pomocí `Enum.TryParse`**

```csharp

string vstup = "Neaktivni";

if (Enum.TryParse(vstup, out Stav stav))

{

    Console.WriteLine("Platný stav: " + stav);

}

else

{

    Console.WriteLine("Neplatná hodnota!");

}

```

✅ `TryParse()` vrací `false`, pokud text **není platný enum**.

---

## **6. `enum` s `Flags` -- Kombinace hodnot**

`Flags` umožňuje **spojit více hodnot dohromady**.

```csharp

[Flags]

enum Prava

{

    Zadna = 0,

    Cteni = 1,

    Zapis = 2,

    Spusteni = 4,

    Vse = Cteni | Zapis | Spusteni

}

class Program

{

    static void Main()

    {

        Prava uzivatel = Prava.Cteni | Prava.Zapis;

        Console.WriteLine(uzivatel); // Výstup: Cteni, Zapis

        bool maZapis = (uzivatel & Prava.Zapis) == Prava.Zapis;

        Console.WriteLine("Má právo zápisu? " + maZapis); // Výstup: True

    }

}

```

📌 **Každá hodnota je mocnina dvojky (`1, 2, 4, 8...`)**, což umožňuje **bitové operace**.

✅ `Prava.Vse` obsahuje všechny tři hodnoty.  

✅ Můžeme zjistit, jestli `uzivatel` obsahuje `Zapis`.

---

## **7. Kdy použít `enum`?**

✅ **Použij `enum`, pokud:**  

- Máme **pevně daný seznam možností**.  

- Hodnoty **nebudeme měnit** (např. dny v týdnu).  

- Chceme **zlepšit čitelnost kódu** (`Den.Pondeli` je jasnější než `0`).

❌ **Nepoužívej `enum`, pokud:**  

- Potřebujeme **složitější chování** (použij `class`).  

- Potřebujeme **přidávat nové hodnoty v runtime** (`enum` je statický).  

- Potřebujeme **vazbu na databázi** (raději použij `Dictionary`).

---

## **8. Shrnutí**

- **`enum`** definuje **pevnou sadu konstantních hodnot**.  

- **Každá hodnota má číslo (`int`)** a lze ji měnit.  

- **Lze převádět `enum` na `int` a zpět**.  

- **`Enum.Parse()` převádí string na `enum`**.  

- **`Flags` umožňuje kombinaci hodnot** (`Cteni | Zapis`).

---

🔹 **Další krok:** **Anonymní typy a `Tuple` -- Jak vracet více hodnot z metody! 🚀**