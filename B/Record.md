# Záznamy (`record`)

`record` je speciální typ v C# pro **nezměnitelné datové objekty**. Automaticky generuje porovnání hodnot, `ToString()` a další.

## Základní použití

```csharp
record Osoba(string Jmeno, int Vek);

Osoba o1 = new Osoba("Petr", 30);
Osoba o2 = new Osoba("Petr", 30);

Console.WriteLine(o1 == o2);   // True  (porovnává hodnoty, ne odkaz)
Console.WriteLine(o1);         // Osoba { Jmeno = Petr, Vek = 30 }
```

## Kopírování s úpravou (`with`)

```csharp
Osoba o1 = new Osoba("Petr", 30);
Osoba o2 = o1 with { Vek = 31 };

Console.WriteLine(o1); // Osoba { Jmeno = Petr, Vek = 30 }
Console.WriteLine(o2); // Osoba { Jmeno = Petr, Vek = 31 }
```

`with` vytvoří novou kopii – původní objekt zůstává nezměněný.

## `record class` vs. `record struct`

```csharp
record class  OsobaRef(string Jmeno, int Vek);   // referenční typ (heap)
record struct OsobaVal(string Jmeno, int Vek);   // hodnotový typ (stack)
```

| | `record class` | `record struct` |
|---|---|---|
| Paměť | heap | stack |
| Dědičnost | ano | ne |
| Vhodné pro | DTO, modely API | malá nezměnitelná data |

## `record` vs. `class`

| | `record` | `class` |
|---|---|---|
| Porovnání `==` | podle hodnot | podle odkazu |
| `ToString()` | automatický | musíš napsat |
| Změnitelnost | `init` (volitelně) | libovolná |
| Vhodné pro | DTO, nezměnitelná data | objekty s logikou |

---

<details>
<summary>Volitelné: Dědičnost recordů a `init`</summary>

```csharp
record Osoba(string Jmeno);
record Student(string Jmeno, string Skola) : Osoba(Jmeno);

Student s = new Student("Petr", "Gymnázium");
Console.WriteLine(s); // Student { Jmeno = Petr, Skola = Gymnázium }
```

Vlastnosti `record` jsou ve výchozím stavu nastavené jako `init` – lze je nastavit pouze při vytvoření:

```csharp
record Osoba(string Jmeno, int Vek);
var o = new Osoba("Petr", 30);
// o.Jmeno = "Anna"; // Chyba! Nelze změnit.
```

</details>

---

**Další krok:** [Anonymní typy a Tuples](Anonymni_typy_tuple.md)
