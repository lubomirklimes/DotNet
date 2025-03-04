# Práce s datem a časem

## Základní práce s `DateTime`

```csharp
DateTime ted = DateTime.Now;           // aktuální čas + datum
DateTime dnes = DateTime.Today;        // dnešní datum (bez času)
DateTime utc = DateTime.UtcNow;        // UTC čas

Console.WriteLine(ted);               // 15.06.2025 14:30:00
Console.WriteLine(ted.Year);          // 2025
Console.WriteLine(ted.DayOfWeek);     // Sunday
```

## Vytvoření data

```csharp
DateTime narozeniny = new DateTime(2000, 6, 15);
DateTime s_casem   = new DateTime(2000, 6, 15, 9, 30, 0); // 9:30:00
```

## Aritmetika s datumy

```csharp
DateTime zitra = DateTime.Today.AddDays(1);
DateTime za_mesic = DateTime.Today.AddMonths(1);

TimeSpan rozdil = DateTime.Today - narozeniny;
Console.WriteLine($"Věk ve dnech: {rozdil.Days}");
```

## Formátování

```csharp
DateTime dt = new DateTime(2025, 6, 15, 14, 30, 0);

Console.WriteLine(dt.ToString("d"));               // 15.06.2025
Console.WriteLine(dt.ToString("t"));               // 14:30
Console.WriteLine(dt.ToString("yyyy-MM-dd"));      // 2025-06-15
Console.WriteLine(dt.ToString("HH:mm:ss"));        // 14:30:00
```

## Parsování textu na datum

```csharp
DateTime dt = DateTime.Parse("2025-06-15");

if (DateTime.TryParse("neplatné", out DateTime result))
    Console.WriteLine(result);
else
    Console.WriteLine("Neplatný formát data");
```

---

<details>
<summary>Volitelné: `DateOnly`, `TimeOnly` a časové zóny</summary>

Od .NET 6 existují specializované typy:

```csharp
DateOnly datum = new DateOnly(2025, 6, 15);
TimeOnly cas   = new TimeOnly(14, 30);
```

Pro práci s časovými zónami použij `DateTimeOffset`:

```csharp
DateTimeOffset dto = DateTimeOffset.UtcNow;
DateTimeOffset local = dto.ToLocalTime();
```

</details>

---

**Další krok:** [Základní kolekce a jejich použití](Collections.md)
