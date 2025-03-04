3\. Záznamy (`record`) -- Immutabilní datové typy
================================================

`record` je speciální typ v C#, který umožňuje snadno vytvářet **neměnné** (`immutable`) objekty. Používá se hlavně pro **datové struktury**, kde chceme zachovat hodnoty beze změny.

---

## **1. Co je `record`?**

`record` je podobný `class`, ale má **automatické porovnávání hodnot** a podporuje **neměnnost (`immutability`)**.

📌 **Výhody `record`:**  

✅ **Automatická porovnání hodnot** -- nemusíme přepisovat `Equals()`.  

✅ **Podpora `with` pro snadné kopírování objektů**.  

✅ **Implicitně `sealed`** -- nelze dědit z `record class`, pokud nepoužijeme `record struct`.

---

## **2. Základní použití `record`**

```csharp

record Osoba(string Jmeno, int Vek);

```

Toto je **ekvivalent**:

```csharp

class Osoba

{

    public string Jmeno { get; init; }

    public int Vek { get; init; }

    public Osoba(string jmeno, int vek)

    {

        Jmeno = jmeno;

        Vek = vek;

    }

}

```

📌 **Rozdíly:**  

- `record` **automaticky generuje konstruktor a properties**.  

- `init` znamená, že hodnotu lze **nastavit pouze při vytvoření**.

---

## **3. Jak používat `record`?**

```csharp

using System;

record Osoba(string Jmeno, int Vek);

class Program

{

    static void Main()

    {

        Osoba osoba1 = new Osoba("Petr", 30);

        Osoba osoba2 = new Osoba("Petr", 30);

        Console.WriteLine(osoba1 == osoba2); // True (hodnotové porovnání)

        Console.WriteLine(osoba1); // Osoba { Jmeno = Petr, Vek = 30 }

    }

}

```

✅ `record` porovnává **hodnoty**, ne odkazy!  

✅ `record` generuje **automatický `ToString()`**.

---

## **4. `with` -- Kopírování s úpravou hodnot**

Chceme vytvořit **novou kopii objektu s jinými hodnotami**?

```csharp

record Osoba(string Jmeno, int Vek);

class Program

{

    static void Main()

    {

        Osoba osoba1 = new Osoba("Petr", 30);

        Osoba osoba2 = osoba1 with { Vek = 31 };

        Console.WriteLine(osoba1); // Osoba { Jmeno = Petr, Vek = 30 }

        Console.WriteLine(osoba2); // Osoba { Jmeno = Petr, Vek = 31 }

    }

}

```

📌 **Klíčové slovo `with` vytváří nový objekt, nezmění původní!**

---

## **5. `record class` vs. `record struct`**

🔹 **`record class`** -- Standardní `record` (referenční typ).  

🔹 **`record struct`** -- Hodnotový `record`, jako `struct`.

```csharp

record class OsobaClass(string Jmeno, int Vek); // Reference type

record struct OsobaStruct(string Jmeno, int Vek); // Value type

```

### **Rozdíly mezi `record class` a `record struct`**

| Vlastnost | `record class` | `record struct` |

|-----------|---------------|----------------|

| Typ | Referenční (`reference type`) | Hodnotový (`value type`) |

| Ukládání | Heap (hromada) | Stack (zásobník) |

| Porovnání | Hodnotové (`Equals()`) | Hodnotové (`Equals()`) |

| Kopírování | Odkaz | Kopírování hodnot |

| Dědičnost | ✅ Ano | 🚫 Ne |

📌 **Kdy použít `record struct`?**  

- Pokud pracujeme s **malými datovými objekty**.  

- Pokud chceme **lepší výkon** (méně alokací na heap).

---

## **6. Dědičnost v `record`**

`record` podporuje dědičnost **jen u referenčních typů (`record class`)**.

```csharp

record Osoba(string Jmeno);

record Student(string Jmeno, string Skola) : Osoba(Jmeno);

class Program

{

    static void Main()

    {

        Student s = new Student("Petr", "Gymnázium");

        Console.WriteLine(s); // Student { Jmeno = Petr, Skola = Gymnázium }

    }

}

```

✅ **`record` umí dědit, ale jen jako `class`!**  

❌ `record struct` nemůže dědit od jiného `struct`.

---

## **7. `readonly record struct` -- Immutable `struct`**

Pokud potřebujeme **hodnotový typ, který je neměnný**, použijeme:

```csharp

readonly record struct Bod(int X, int Y);

```

📌 **Všechny properties jsou automaticky `readonly`!**  

📌 **Lze použít jen `record struct`, ne `record class`**.

---

## **8. `ToString()`, `Equals()` a porovnávání**

Všechny `record` automaticky implementují:  

- `ToString()` -- vypíše hodnoty.  

- `Equals()` -- porovnává hodnoty.  

- `GetHashCode()` -- generuje hash podle hodnot.

```csharp

record Osoba(string Jmeno, int Vek);

class Program

{

    static void Main()

    {

        Osoba o1 = new Osoba("Petr", 30);

        Osoba o2 = new Osoba("Petr", 30);

        Console.WriteLine(o1.Equals(o2)); // True

        Console.WriteLine(o1.GetHashCode()); // Např. 1234567

    }

}

```

📌 **Porovnávání `record` funguje podle hodnot, ne podle referencí!**

---

## **9. Kdy použít `record` místo `class`?**

✅ **Použij `record`, pokud:**  

- Data jsou **neměnná (immutable)**.  

- Potřebujeme **hodnotové porovnávání** (`Equals()`, `==`).  

- Používáme **kopírování s `with`**.  

- Pracujeme s **datovými modely** (např. DTO, zprávy v API).

❌ **Použij `class`, pokud:**  

- Objekt má složité **chování** (metody, logiku).  

- Potřebujeme **mutable objekty**.  

- Potřebujeme **dědičnost mezi běžnými třídami**.

---

## **10. Shrnutí**

- **`record`** je **speciální datový typ**, který je **neměnný (`immutable`)**.  

- **Automatické porovnání hodnot** -- nemusíme přepisovat `Equals()`.  

- **Podporuje `with` pro kopírování objektů**.  

- **`record class` (referenční) vs. `record struct` (hodnotový)**.  

- **`readonly record struct`** pro **neměnné struktury**.  

- **Vhodné pro DTO, modely a neměnná data**.

---

🔹 **Další krok:** **Výčtové typy (`enum`) -- Jak pracovat s konstantními hodnotami! 🚀**