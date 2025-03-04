# Řetězce a jejich zpracování

`string` je nezmměnitelný (immutable) řetězec znaků. Každá úprava vytvoří nový řetězec.

## Základní operace

```csharp
string s = "Ahoj, světe!";

Console.WriteLine(s.Length);              // 13
Console.WriteLine(s.ToUpper());           // AHOJ, SVĚTe!
Console.WriteLine(s.ToLower());           // ahoj, světe!
Console.WriteLine(s.Contains("svět")); // True
Console.WriteLine(s.Replace("Ahoj", "Nazdar")); // Nazdar, světe!
Console.WriteLine(s.Trim());              // odstraní mezery na krajich
```

## Rozdělování a spojování

```csharp
string csv = "Petr;Anna;Karel";
string[] jmena = csv.Split(';');          // ["Petr", "Anna", "Karel"]

string spojena = string.Join(", ", jmena); // "Petr, Anna, Karel"
```

## Interpolace a formátování

```csharp
string jmeno = "Petr";
int vek = 30;

// Interpolace (doporučené)
Console.WriteLine($"Jméno: {jmeno}, věk: {vek}");

// Formátování čísel
double cena = 1234.5;
Console.WriteLine($"{cena:F2}");   // 1234.50
Console.WriteLine($"{cena:N0}");   // 1 235  (oddělení tisíců)
```

## Vyhledávání a řez

```csharp
string text = "Hello, World!";

int index = text.IndexOf("World");   // 7
string cast = text.Substring(7, 5);  // World
string cast2 = text[7..12];          // World  (range operátor, moderní C#)

bool zacina = text.StartsWith("Hello"); // True
bool konci  = text.EndsWith("!");       // True
```

---

<details>
<summary>Volitelné: `StringBuilder` pro časté skládání řetězců</summary>

Při skládání mnoha řetězců ve smyčce je `string +` neefektivní (vytváří novou kopii při každém spočení). Použij `StringBuilder`:

```csharp
using System.Text;

StringBuilder sb = new StringBuilder();
for (int i = 0; i < 1000; i++)
    sb.Append($"Radek {i}\n");

string vysledek = sb.ToString();
```

</details>

---

**Další krok:** [Výjimky a jejich ošetření](Exceptions.md)
