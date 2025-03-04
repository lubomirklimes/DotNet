7\. Objektově orientované programování (OOP)
===========================================

OOP je způsob programování, kde si program rozdělujeme na **objekty**, které mají **vlastnosti** a **chování**.

---

## **1. Co je objekt a třída?**

### **Třída**  

Třída je **šablona**, podle které se vytvářejí objekty.  

Třída popisuje, **jaké vlastnosti** a **jaké metody** objekt bude mít.

### **Objekt**  

Objekt je **konkrétní instance třídy**. Každý objekt může mít různé hodnoty vlastností.

---

## **2. Vytvoření třídy a objektu**  

Vytvoříme třídu `Auto`, která bude mít vlastnosti **značka** a **rok výroby** a metodu **Start()**.

```csharp

using System;

class Auto

{

    public string Znacka;  // Vlastnost

    public int RokVyroby;  // Vlastnost

    public void Start()  // Metoda

    {

        Console.WriteLine(Znacka + " startuje!");

    }

}

class Program

{

    static void Main()

    {

        Auto mojeAuto = new Auto(); // Vytvoření objektu

        mojeAuto.Znacka = "Škoda";

        mojeAuto.RokVyroby = 2020;

        Console.WriteLine("Auto: " + mojeAuto.Znacka + ", rok výroby: " + mojeAuto.RokVyroby);

        mojeAuto.Start(); // Zavolání metody objektu

    }

}

```

✅ **Výstup:**  

```

Auto: Škoda, rok výroby: 2020  

Škoda startuje!  

```

📌 **Poznámka:**  

- `public` znamená, že vlastnost nebo metoda je **přístupná zvenku**.  

- `mojeAuto` je **objekt** vytvořený podle třídy `Auto`.

---

## **3. Konstruktor -- Automatické nastavení hodnot**  

Konstruktor je speciální metoda, která se spustí **při vytváření objektu** a nastaví vlastnosti.

```csharp

using System;

class Auto

{

    public string Znacka;

    public int RokVyroby;

    // Konstruktor

    public Auto(string znacka, int rok)

    {

        Znacka = znacka;

        RokVyroby = rok;

    }

    public void Start()

    {

        Console.WriteLine(Znacka + " startuje!");

    }

}

class Program

{

    static void Main()

    {

        Auto mojeAuto = new Auto("Škoda", 2020); // Použití konstruktoru

        Console.WriteLine("Auto: " + mojeAuto.Znacka + ", rok výroby: " + mojeAuto.RokVyroby);

        mojeAuto.Start();

    }

}

```

✅ **Výstup:**  

```

Auto: Škoda, rok výroby: 2020  

Škoda startuje!  

```

📌 **Konstruktor** se automaticky spustí při vytvoření objektu a nastaví vlastnosti.

---

## **4. Vlastnosti (Properties) -- Lepší správa dat**  

Vlastnosti umožňují **kontrolovat přístup** k hodnotám.

### **Příklad: Použití `get` a `set`**

```csharp

using System;

class Osoba

{

    private string jmeno; // Privátní proměnná

    public string Jmeno

    {

        get { return jmeno; } // Získání hodnoty

        set { jmeno = value; } // Nastavení hodnoty

    }

}

class Program

{

    static void Main()

    {

        Osoba osoba = new Osoba();

        osoba.Jmeno = "Petr"; // Použití vlastnosti

        Console.WriteLine("Jméno: " + osoba.Jmeno);

    }

}

```

✅ **Výstup:** `Jméno: Petr`

📌 **Výhody properties:**  

- **Můžeme přidat podmínky pro nastavení hodnoty.**  

- **Lze zabránit přímé změně proměnných.**

---

## **5. Dědičnost -- Když jedna třída vychází z jiné**  

Dědičnost umožňuje **vytvořit novou třídu**, která vychází z existující třídy.

### **Příklad: Třída `Zvire` a její potomek `Pes`**

```csharp

using System;

class Zvire

{

    public string Jmeno;

    public void Zvuk()

    {

        Console.WriteLine(Jmeno + " vydává zvuk.");

    }

}

class Pes : Zvire // Pes dědí od třídy Zvire

{

    public void Stekej()

    {

        Console.WriteLine(Jmeno + " štěká!");

    }

}

class Program

{

    static void Main()

    {

        Pes mujPes = new Pes();

        mujPes.Jmeno = "Rex";

        mujPes.Zvuk(); // Metoda z třídy Zvire

        mujPes.Stekej(); // Metoda ze třídy Pes

    }

}

```

✅ **Výstup:**  

```

Rex vydává zvuk.  

Rex štěká!  

```

📌 **Dědičnost šetří kód**, protože `Pes` zdědil metodu `Zvuk()` z třídy `Zvire`.

---

## **6. Překrývání metod (`override`)**  

Díky `override` můžeme **změnit chování metody** z rodičovské třídy.

### **Příklad: Překrytí metody `Zvuk()`**

```csharp

using System;

class Zvire

{

    public virtual void Zvuk() // Virtuální metoda

    {

        Console.WriteLine("Zvire vydává zvuk.");

    }

}

class Pes : Zvire

{

    public override void Zvuk() // Přepsaná metoda

    {

        Console.WriteLine("Pes štěká!");

    }

}

class Program

{

    static void Main()

    {

        Pes mujPes = new Pes();

        mujPes.Zvuk(); // Volá přepsanou metodu

    }

}

```

✅ **Výstup:** `Pes štěká!`

📌 **Klíčová slova:**  

- `virtual` -- označuje metodu, která se dá přepsat.  

- `override` -- přepisuje metodu v potomkovi.

---

## **7. Rozhraní (`interface`) -- Smlouva pro třídy**  

Rozhraní říká, **jaké metody musí třída obsahovat**, ale neříká **jak fungují**.

### **Příklad: Rozhraní `IZvire`**

```csharp

using System;

interface IZvire

{

    void Zvuk(); // Pouze deklarace metody

}

class Pes : IZvire

{

    public void Zvuk()

    {

        Console.WriteLine("Pes štěká!");

    }

}

class Program

{

    static void Main()

    {

        Pes mujPes = new Pes();

        mujPes.Zvuk();

    }

}

```

✅ **Výstup:** `Pes štěká!`

📌 **Rozhraní je jako plán**, který musí každá třída splnit.

---

## **Shrnutí**  

- **Třída** je šablona pro objekt.  

- **Objekt** je konkrétní instance třídy.  

- **Konstruktor** nastavuje vlastnosti při vytvoření objektu.  

- **Properties (`get`, `set`)** kontrolují přístup k hodnotám.  

- **Dědičnost** umožňuje třídám přebírat vlastnosti jiné třídy.  

- **Override** přepisuje metody.  

- **Rozhraní (`interface`)** určuje, jaké metody musí třída obsahovat.

🔹 **Další krok:** **Výčtové typy (`enum`) -- Jak pracovat s konstantními hodnotami! 🚀**
