4. Smyčky a opakování
=====================

Smyčky umožňují opakovat stejný kód vícekrát, což šetří čas a zjednodušuje programy.

---

### **1. FOR -- Smyčka s pevným počtem opakování**  

Používáme ji, když víme, kolikrát se má kód opakovat.

#### **Základní syntaxe:**

```csharp

for (počáteční_hodnota; podmínka; změna)

{

    // Kód, který se opakuje

}

```

#### **Příklad: Výpis čísel od 1 do 5**

```csharp

using System;

class Program

{

    static void Main()

    {

        for (int i = 1; i <= 5; i++)

        {

            Console.WriteLine("Číslo: " + i);

        }

    }

}

```

✅ Program vypíše:  

```

Číslo: 1  

Číslo: 2  

Číslo: 3  

Číslo: 4  

Číslo: 5  

```

---

### **2. WHILE -- Smyčka, která běží, dokud je splněna podmínka**  

Používáme ji, když **nevíme přesný počet opakování**.

#### **Základní syntaxe:**

```csharp

while (podmínka)

{

    // Kód se opakuje, dokud je podmínka splněna

}

```

#### **Příklad: Počítadlo do 5**

```csharp

using System;

class Program

{

    static void Main()

    {

        int i = 1;

        while (i <= 5)

        {

            Console.WriteLine("Počítám: " + i);

            i++;

        }

    }

}

```

✅ Program vypíše čísla 1 až 5.

📌 **Pozor!** Pokud zapomeneme **zvýšit `i`**, smyčka poběží **nekonečně**.

---

### **3. DO-WHILE -- Smyčka, která se spustí alespoň jednou**  

Podmínka se kontroluje **až po prvním průchodu**, takže kód proběhne minimálně **jednou**.

#### **Základní syntaxe:**

```csharp

do

{

    // Kód se provede alespoň jednou

} while (podmínka);

```

#### **Příklad: Požádáme uživatele o číslo větší než 10**

```csharp

using System;

class Program

{

    static void Main()

    {

        int cislo;

        do

        {

            Console.Write("Zadej číslo větší než 10: ");

            cislo = Convert.ToInt32(Console.ReadLine());

        } while (cislo <= 10);

        Console.WriteLine("Správně! Zadal jsi: " + cislo);

    }

}

```

✅ Pokud uživatel zadá **5**, program ho znovu požádá o číslo.  

✅ Pokud zadá **15**, program pokračuje dál.

---

### **4. BREAK a CONTINUE -- Ovládání smyčky**  

Někdy chceme **ukončit smyčku dříve** nebo **přeskočit část kódu**.

#### **BREAK -- okamžité ukončení smyčky**

```csharp

using System;

class Program

{

    static void Main()

    {

        for (int i = 1; i <= 10; i++)

        {

            if (i == 5)

            {

                Console.WriteLine("Končím u čísla 5!");

                break;

            }

            Console.WriteLine(i);

        }

    }

}

```

✅ Smyčka skončí u **5** a dál nepokračuje.

---

#### **CONTINUE -- přeskočení jedné iterace**

```csharp

using System;

class Program

{

    static void Main()

    {

        for (int i = 1; i <= 5; i++)

        {

            if (i == 3)

            {

                continue; // Přeskočí číslo 3

            }

            Console.WriteLine(i);

        }

    }

}

```

✅ Program vypíše: **1, 2, 4, 5** (číslo **3** přeskočí).

---

### **5. Vnořené smyčky -- Smyčka v smyčce**  

Používáme, když chceme **opakovat kód uvnitř jiné smyčky**.

#### **Příklad: Tabulka násobků**

```csharp

using System;

class Program

{

    static void Main()

    {

        for (int i = 1; i <= 3; i++)

        {

            for (int j = 1; j <= 3; j++)

            {

                Console.Write(i * j + "\t");

            }

            Console.WriteLine();

        }

    }

}

```

✅ Program vypíše násobilku pro čísla **1 až 3**:  

```

1   2   3  

2   4   6  

3   6   9  

```

---

### **Shrnutí**  

- **`for`** -- když známe počet opakování.  

- **`while`** -- když nevíme přesný počet opakování.  

- **`do-while`** -- když se musí kód provést **alespoň jednou**.  

- **`break`** -- ukončí smyčku.  

- **`continue`** -- přeskočí jednu iteraci.  

- **Vnořené smyčky** -- smyčka uvnitř smyčky.

🔹 **Další krok:** Naučíme se, jak psát vlastní funkce a metody! 🚀