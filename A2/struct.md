2.\ Struktury (`struct`) a rozdíl oproti třídám
==============================================

Struktury (`struct`) v C# jsou podobné třídám (`class`), ale mají několik důležitých rozdílů. Používají se hlavně pro **malá, neměnná data**, která nepotřebují složité chování.

---

## **1. Co je `struct`?**  

Struktura (`struct`) je **hodnotový typ**, což znamená, že se **ukládá přímo do paměti** a ne na heap (hromadu).

📌 **Kdy použít `struct`?**  

✅ Když potřebujeme **rychlý a lehký objekt**.  

✅ Když **hodnoty nebudeme měnit** (např. souřadnice bodu).  

✅ Když chceme **vyhnout se garbage collection** (struktury se automaticky uvolňují).

---

## **2. Základní syntaxe `struct`**

```csharp

struct Bod

{

    public int X;

    public int Y;

    public Bod(int x, int y)

    {

        X = x;

        Y = y;

    }

}

```

📌 **Poznámky:**  

- `struct` obsahuje **pole** (`X`, `Y`) a **konstruktor**.  

- Struktura **nemůže mít prázdný konstruktor** (C# automaticky vytváří výchozí konstruktor).

---

## **3. Jak se používá `struct`?**

```csharp

using System;

struct Bod

{

    public int X;

    public int Y;

    public Bod(int x, int y)

    {

        X = x;

        Y = y;

    }

    public void Vypis()

    {

        Console.WriteLine($"Bod má souřadnice ({X}, {Y})");

    }

}

class Program

{

    static void Main()

    {

        Bod b1 = new Bod(10, 20);

        b1.Vypis(); // Výstup: Bod má souřadnice (10, 20)

    }

}

```

📌 **Poznámky:**  

- `Bod b1` se vytváří **bez použití `new`** (ale můžeme `new` použít).  

- **Struktury se kopírují při přiřazení** (viz níže).

---

## **4. `struct` vs. `class` -- Jaký je rozdíl?**

| **Vlastnost** | **`struct` (struktura)** | **`class` (třída)** |

|--------------|----------------|----------------|

| **Typ** | Hodnotový (`value type`) | Referenční (`reference type`) |

| **Ukládání v paměti** | Stack (zásobník) | Heap (hromada) |

| **Kopírování při přiřazení** | Vytváří kopii | Přiřazuje odkaz |

| **Dědičnost (`inheritance`)** | 🚫 Ne | ✅ Ano |

| **Garbage Collection (GC)** | 🚫 Neovlivňuje | ✅ Uvolňuje se pomocí GC |

| **Použití** | Malé, neměnné datové objekty | Velké a složité objekty |

---

## **5. Kopírování `struct` vs. `class`**

Když přiřadíme strukturu, vytvoří se **kopie**, zatímco u třídy se předá **odkaz**.

### **Příklad: `struct` vytváří kopii**  

```csharp

struct Bod

{

    public int X;

    public int Y;

}

class Program

{

    static void Main()

    {

        Bod b1 = new Bod { X = 5, Y = 10 };

        Bod b2 = b1; // Kopírování struktury

        b2.X = 100; // Změní se jen b2

        Console.WriteLine("b1: " + b1.X); // Výstup: 5

        Console.WriteLine("b2: " + b2.X); // Výstup: 100

    }

}

```

✅ `b1` zůstane **nezměněný**, protože `struct` vytváří kopii.

### **Příklad: `class` předává odkaz**  

```csharp

class Bod

{

    public int X;

    public int Y;

}

class Program

{

    static void Main()

    {

        Bod b1 = new Bod { X = 5, Y = 10 };

        Bod b2 = b1; // Odkaz na stejný objekt

        b2.X = 100; // Změní se i b1

        Console.WriteLine("b1: " + b1.X); // Výstup: 100

        Console.WriteLine("b2: " + b2.X); // Výstup: 100

    }

}

```

✅ `b1` **se změní**, protože `class` předává odkaz na stejný objekt.

---

## **6. `readonly struct` -- Proč ho používat?**  

`readonly struct` zajišťuje, že se hodnoty **nemohou změnit** po vytvoření.

```csharp

readonly struct Bod

{

    public int X { get; }

    public int Y { get; }

    public Bod(int x, int y)

    {

        X = x;

        Y = y;

    }

}

```

📌 **Použití `readonly` znamená:**  

- Každá vlastnost **musí mít `get;`, ale ne `set;`**.  

- Struktura **je neměnná (immutable)**.

---

## **7. `ref struct` -- Optimalizace výkonu**  

`ref struct` znamená, že struktura **může být uložena pouze na stacku**. To **zabraňuje garbage collection** a je vhodné pro dočasná data.

```csharp

ref struct SpanWrapper

{

    private Span<int> data;

    public SpanWrapper(Span<int> span)

    {

        data = span;

    }

}

```

📌 `ref struct` **nemůže být součástí `List<T>` nebo `class`**, protože se musí **uchovávat pouze na stacku**.

---

## **8. Kdy použít `struct` místo `class`?**

✅ **Použij `struct`, pokud:**  

- **Objekt je malý** a má **méně než 16 bajtů** (např. `Point`, `Rectangle`).  

- **Hodnoty se nemění po vytvoření** (immutable).  

- **Nechceme alokaci na heap** (lepší výkon).  

- **Nechceme dědičnost** (struktury nemají dědičnost).

❌ **Použij `class`, pokud:**  

- **Objekt má složité chování** (např. obsahuje metody pro zpracování dat).  

- **Potřebujeme dědičnost**.  

- **Velké objekty** -- uložení na heap je efektivnější.

---

## **Shrnutí**  

- **`struct`** je **hodnotový typ**, zatímco `class` je **referenční typ**.  

- **`struct` se kopíruje** při přiřazení, `class` předává odkaz.  

- **`readonly struct`** zajistí, že hodnoty **nelze měnit**.  

- **`ref struct`** umožňuje **vysoký výkon** bez garbage collection.  

- **Struktury jsou vhodné pro malé a neměnné objekty**.

🔹 **Další krok:** **Záznamy (`record`) -- Immutabilní datové typy**! 🚀