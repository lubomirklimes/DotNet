4\. Anonymní typy a `Tuple` -- Jak vracet více hodnot z metody
==============================================================

V C# můžeme **seskupit více hodnot dohromady** pomocí anonymních typů a `Tuple`. Díky nim nemusíme vždy vytvářet nové třídy nebo struktury.

---

## **1. Anonymní typy -- Rychlé seskupení dat**

Anonymní typy umožňují **vytvářet objekty bez nutnosti definovat třídu**.

### **Základní příklad anonymního typu**

```csharp

var osoba = new { Jmeno = "Petr", Vek = 30 };

Console.WriteLine($"Jméno: {osoba.Jmeno}, Věk: {osoba.Vek}");

// Výstup: Jméno: Petr, Věk: 30

```

✅ **`var` se použije automaticky**  

✅ **`osoba` má vlastnosti `Jmeno` a `Vek`**

📌 **Anonymní typy jsou `readonly` -- hodnoty nelze měnit po vytvoření!**

---

## **2. Použití anonymních typů v kolekcích**

```csharp

var seznam = new[]

{

    new { Jmeno = "Petr", Vek = 30 },

    new { Jmeno = "Anna", Vek = 25 }

};

foreach (var osoba in seznam)

{

    Console.WriteLine($"{osoba.Jmeno}, {osoba.Vek} let");

}

```

✅ **Můžeme vytvořit pole anonymních objektů**  

✅ **Skvělé pro dočasná data (např. z databáze)**

📌 **Anonymní typy se nedají vracet z metod -- použijme `Tuple`!**

---

## **3. `Tuple` -- Vrácení více hodnot z metody**

`Tuple` je **datová struktura**, která **uchovává více hodnot různého typu**.

### **Základní příklad `Tuple`**

```csharp

var osoba = Tuple.Create("Petr", 30);

Console.WriteLine($"Jméno: {osoba.Item1}, Věk: {osoba.Item2}");

// Výstup: Jméno: Petr, Věk: 30

```

✅ **Každý prvek `Tuple` má číslo (`Item1`, `Item2`, ...)**

📌 **Nevýhoda:** `Item1`, `Item2` nejsou popisné -- lepší použít `ValueTuple`!

---

## **4. `ValueTuple` -- Lepší verze `Tuple` s názvy**

```csharp

(string jmeno, int vek) osoba = ("Petr", 30);

Console.WriteLine($"Jméno: {osoba.jmeno}, Věk: {osoba.vek}");

// Výstup: Jméno: Petr, Věk: 30

```

✅ **Každý prvek má vlastní jméno**

📌 **`ValueTuple` je lehčí a rychlejší než `Tuple`**

---

## **5. Vrácení více hodnot z metody pomocí `Tuple`**

```csharp

static (string, int) ZiskejOsobu()

{

    return ("Petr", 30);

}

class Program

{

    static void Main()

    {

        var osoba = ZiskejOsobu();

        Console.WriteLine($"Jméno: {osoba.Item1}, Věk: {osoba.Item2}");

    }

}

```

✅ **Metoda vrací dvě hodnoty bez nutnosti definovat třídu**

📌 **Lepší použít pojmenované hodnoty!**

---

## **6. Pojmenované hodnoty v `Tuple`**

```csharp

static (string jmeno, int vek) ZiskejOsobu()

{

    return ("Petr", 30);

}

class Program

{

    static void Main()

    {

        var osoba = ZiskejOsobu();

        Console.WriteLine($"Jméno: {osoba.jmeno}, Věk: {osoba.vek}");

    }

}

```

✅ **Přehlednější než `Item1`, `Item2`**

---

## **7. Dekonstrukce `Tuple` -- Přiřazení hodnot**

```csharp

(string jmeno, int vek) = ZiskejOsobu();

Console.WriteLine($"Jméno: {jmeno}, Věk: {vek}");

```

✅ **Přímé rozdělení hodnot do proměnných**

📌 **Můžeme ignorovat hodnotu pomocí `_`**

```csharp

(_, int vek) = ZiskejOsobu();

Console.WriteLine($"Věk: {vek}");

```

✅ **Použijeme jen část `Tuple`**

---

## **8. Kdy použít `Tuple` místo třídy?**

✅ **Použij `Tuple`, pokud:**  

- Potřebujeme **vrátit více hodnot z metody**.  

- Data jsou **krátkodobá a nepotřebují chování** (např. metody).  

- Nechceme vytvářet **zbytečnou třídu**.

❌ **Použij `class` nebo `record`, pokud:**  

- Potřebujeme **složitější logiku**.  

- Chceme **dědičnost nebo metody**.  

- Objekt se **bude často měnit**.

---

## **9. Shrnutí**

- **Anonymní typy (`new {}`)** umožňují **rychlé seskupení dat**.  

- **`Tuple` (`Tuple.Create`)** umožňuje vracet více hodnot, ale používá **Item1, Item2**.  

- **`ValueTuple` (`(string, int)`)** je **rychlejší** a podporuje **pojmenované hodnoty**.  

- **Metody mohou vracet `Tuple`** a hodnoty lze **dekódovat** (`(x, y) = metoda();`).

---

🔹 **Další krok:** **Delegáty a události (`delegate`, `event`) -- Jak na zpětná volání! 🚀**