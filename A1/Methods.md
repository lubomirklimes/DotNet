5\. Funkce a metody
==================

Funkce (v C# nazývané **metody**) pomáhají rozdělit program na menší části, které provádějí konkrétní úkoly. Díky nim je kód přehlednější a snadněji se upravuje.

---

### **1. Co je metoda?**  

Metoda je **skupina příkazů**, které se vykonají, když metodu zavoláme.

#### **Základní struktura metody:**

```csharp

návratový_typ NázevMetody(parametry)

{

    // Kód, který metoda provede

}

```

#### **Příklad: Metoda, která vypíše text**

```csharp

using System;

class Program

{

    static void Main()

    {

        Pozdrav();

    }

    static void Pozdrav()

    {

        Console.WriteLine("Ahoj, vítám tě v programu!");

    }

}

```

✅ Výstup: `Ahoj, vítám tě v programu!`  

📌 **Poznámka:** `Main()` je hlavní metoda, která se spustí při startu programu.

---

### **2. Metody s parametry**  

Metoda může přijímat **parametry**, což jsou **vstupní hodnoty**, se kterými pracuje.

#### **Příklad: Metoda, která pozdraví uživatele jménem**

```csharp

using System;

class Program

{

    static void Main()

    {

        Pozdrav("Petr");

        Pozdrav("Anna");

    }

    static void Pozdrav(string jmeno)

    {

        Console.WriteLine("Ahoj, " + jmeno + "!");

    }

}

```

✅ Výstup:  

```

Ahoj, Petr!  

Ahoj, Anna!  

```

📌 **Poznámka:** Parametr `jmeno` umožňuje použít stejnou metodu pro různé lidi.

---

### **3. Metody s návratovou hodnotou**  

Metoda může **vracet hodnotu**, kterou pak můžeme použít v programu.

#### **Příklad: Metoda, která sčítá dvě čísla**

```csharp

using System;

class Program

{

    static void Main()

    {

        int soucet = Secti(5, 3);

        Console.WriteLine("Součet: " + soucet);

    }

    static int Secti(int a, int b)

    {

        return a + b;

    }

}

```

✅ Výstup: `Součet: 8`  

📌 **Poznámka:** `return` vrací výsledek výpočtu.

---

### **4. Přetížení metod**  

Metody mohou mít **stejný název**, pokud mají různé **parametry**.

#### **Příklad: Pozdrav s jedním a dvěma parametry**

```csharp

using System;

class Program

{

    static void Main()

    {

        Pozdrav();

        Pozdrav("Petr");

    }

    static void Pozdrav()

    {

        Console.WriteLine("Ahoj, neznámý uživateli!");

    }

    static void Pozdrav(string jmeno)

    {

        Console.WriteLine("Ahoj, " + jmeno + "!");

    }

}

```

✅ Výstup:  

```

Ahoj, neznámý uživateli!  

Ahoj, Petr!  

```

📌 **Poznámka:** C# sám pozná, kterou metodu použít podle počtu argumentů.

---

### **5. Rekurzivní metody**  

Metoda může volat **sama sebe**. To se hodí např. pro výpočet faktoriálu.

#### **Příklad: Faktoriál čísla**

```csharp

using System;

class Program

{

    static void Main()

    {

        Console.WriteLine("Faktoriál 5 je: " + Faktorial(5));

    }

    static int Faktorial(int n)

    {

        if (n == 1)

            return 1;

        return n * Faktorial(n - 1);

    }

}

```

✅ Výstup: `Faktoriál 5 je: 120`

---

### **Shrnutí**  

- **Metody usnadňují opakované části kódu.**  

- **Metody mohou přijímat parametry.**  

- **Metody mohou vracet hodnotu (`return`).**  

- **Přetížení metod umožňuje použít stejný název s různými parametry.**  

- **Rekurze umožňuje metodě volat sama sebe.**

🔹 **Další krok:** Naučíme se pracovat s poli a seznamy! 🚀