# Práce se soubory a adresáři

.NET nabízí třídy `File`, `Directory` a `Path` pro práci se souborovým systémem.

## Čtení a zápis souboru

```csharp
// Zápis celého textu
File.WriteAllText("pozdrav.txt", "Ahoj světe!");

// Čtení celého textu
string obsah = File.ReadAllText("pozdrav.txt");
Console.WriteLine(obsah); // Ahoj světe!

// Čtení po řádcích
string[] radky = File.ReadAllLines("pozdrav.txt");
foreach (string radek in radky)
    Console.WriteLine(radek);
```

## Přidání textu a práce s adresáři

```csharp
File.AppendAllText("log.txt", "Nový záznam\n");

Directory.CreateDirectory("moje_slozka");
bool existuje = Directory.Exists("moje_slozka"); // True

string[] soubory = Directory.GetFiles(".", "*.txt"); // všechny .txt v aktu. adresáři
```

## Práce s cestami (`Path`)

```csharp
string cesta = Path.Combine("data", "soubor.txt");   // data\soubor.txt
string nazev = Path.GetFileName(cesta);              // soubor.txt
string ext   = Path.GetExtension(cesta);             // .txt
string adresar = Path.GetDirectoryName(cesta);       // data
```

## Správné čtení velkých souborů po řádcích

```csharp
using StreamReader reader = new StreamReader("velky_soubor.txt");
string? radek;
while ((radek = reader.ReadLine()) != null)
{
    Console.WriteLine(radek);
}
```

`StreamReader` načete soubor postupně – nespotrebuje tolik paměti jako `ReadAllLines`.

---

<details>
<summary>Volitelné: `FileStream` a binární soubory</summary>

```csharp
// Zápis binárních dat
byte[] data = { 0x48, 0x65, 0x6C, 0x6C, 0x6F };
File.WriteAllBytes("data.bin", data);

// Čtení
byte[] nacteno = File.ReadAllBytes("data.bin");
```

Pro velké soubory používej `FileStream` s bufferovaným čtením.

</details>

---

**Další krok:** [Práce s datem a časem](DateTime.md)
