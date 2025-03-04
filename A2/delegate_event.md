5\. Delegáty a události (`delegate`, `event`) -- Jak na zpětná volání
=====================================================================

Delegáty a události umožňují v C# předávat **metody jako parametry** a reagovat na akce.

---

## **1. Co je `delegate`?**

Delegát (`delegate`) je **reference na metodu**. Můžeme pomocí něj uložit **odkaz na metodu** a zavolat ji později.

### **Základní syntaxe delegátu**

```csharp

delegate void MojeMetoda(); // Definice delegátu

class Program

{

    static void Pozdrav()

    {

        Console.WriteLine("Ahoj světe!");

    }

    static void Main()

    {

        MojeMetoda metoda = Pozdrav; // Přiřazení metody

        metoda(); // Volání přes delegát

    }

}

```

✅ **`metoda()` volá `Pozdrav()` přes delegát.**

📌 **Delegát umožňuje předat metodu jako proměnnou!**

---

## **2. Delegát s parametry a návratovou hodnotou**

```csharp

delegate int Operace(int a, int b);

class Program

{

    static int Scitani(int x, int y) => x + y;

    static void Main()

    {

        Operace op = Scitani;

        Console.WriteLine(op(3, 5)); // Výstup: 8

    }

}

```

✅ **Delegát `Operace` odkazuje na `Scitani()`**

📌 **Delegáty umožňují vytvořit generické operace.**

---

## **3. Více metod v delegátu (Multicast Delegate)**

```csharp

delegate void Zprava();

class Program

{

    static void Metoda1() => Console.WriteLine("První metoda");

    static void Metoda2() => Console.WriteLine("Druhá metoda");

    static void Main()

    {

        Zprava zprava = Metoda1;

        zprava += Metoda2; // Přidání další metody

        zprava(); // Volá obě metody

    }

}

```

✅ **Přidání metod pomocí `+=`**  

✅ **Volání delegátu zavolá všechny metody v pořadí**

---

## **4. Použití `Func<T>` a `Action<T>` místo `delegate`**

### **`Func<T>` -- Delegát s návratovou hodnotou**

```csharp

Func<int, int, int> scitani = (a, b) => a + b;

Console.WriteLine(scitani(3, 5)); // Výstup: 8

```

✅ **`Func<T, TResult>` automaticky vrací hodnotu**

---

### **`Action<T>` -- Delegát bez návratové hodnoty**

```csharp

Action<string> vypis = text => Console.WriteLine(text);

vypis("Ahoj!"); // Výstup: Ahoj!

```

✅ **`Action<T>` je pro metody bez návratové hodnoty**

---

## **5. Události (`event`) -- Reakce na akce**

Události (`event`) jsou speciální delegáty používané k **oznámení, že se něco stalo**.

```csharp

using System;

class Tlakomer

{

    public delegate void VysokyTlakHandler();

    public event VysokyTlakHandler VysokyTlak;

    public void Zmerit(int hodnota)

    {

        if (hodnota > 140)

            VysokyTlak?.Invoke(); // Spustí událost

    }

}

class Program

{

    static void Varovani() => Console.WriteLine("Pozor, vysoký tlak!");

    static void Main()

    {

        Tlakomer tlakomer = new Tlakomer();

        tlakomer.VysokyTlak += Varovani; // Přihlášení k události

        tlakomer.Zmerit(150); // Spustí událost a zavolá `Varovani()`

    }

}

```

✅ **`VysokyTlak` se zavolá, když je hodnota vyšší než 140**

📌 **Používá se v GUI (tlačítka), senzorech nebo hrách.**

---

## **6. Předávání parametrů do událostí (`EventHandler`)**

```csharp

using System;

class Stahovani

{

    public event EventHandler<int> PrubehStahovani;

    public void Stahuj()

    {

        for (int i = 0; i <= 100; i += 25)

        {

            PrubehStahovani?.Invoke(this, i); // Oznámení o průběhu

        }

    }

}

class Program

{

    static void ZobrazPrubeh(object sender, int procenta)

    {

        Console.WriteLine($"Staženo: {procenta}%");

    }

    static void Main()

    {

        Stahovani stahovani = new Stahovani();

        stahovani.PrubehStahovani += ZobrazPrubeh; // Přihlášení k události

        stahovani.Stahuj();

    }

}

```

✅ **`PrubehStahovani` posílá informace o průběhu.**

📌 **Používáme `EventHandler<T>` pro události s daty!**

---

## **7. Kdy použít `delegate`, `event`, `Func<T>` a `Action<T>`?**

| Použití | Delegát (`delegate`) | Událost (`event`) | `Func<T>` / `Action<T>` |

|---------|---------------------|------------------|----------------|

| Předávání metody jako parametru | ✅ Ano | ❌ Ne | ✅ Ano |

| Reakce na akce (GUI, senzory) | ❌ Ne | ✅ Ano | ❌ Ne |

| Více metod současně (`+=`) | ✅ Ano | ✅ Ano | ✅ Ano |

| Návratová hodnota | ✅ Ano | ❌ Ne | ✅ Ano (`Func<T>`) |

---

## **8. Shrnutí**

- **`delegate`** -- **odkaz na metodu**, lze předávat jako parametr.  

- **Multicast delegate (`+=`)** -- **lze přidat více metod**.  

- **`Func<T>` a `Action<T>`** -- **zjednodušení delegátů**.  

- **`event`** -- **oznámení, že se něco stalo**.  

- **`EventHandler<T>`** -- **předávání dat událostí**.

---

🔹 **Další krok:** **LINQ -- Jak efektivně pracovat s kolekcemi! 🚀**