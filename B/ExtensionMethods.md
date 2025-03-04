# Extension methods

Extension methods umožňují **přidávat metody k existujícím typům** bez dědičnosti nebo úpravy původního kódu. Jsou základem LINQ a fluent API.

## 1. Koncept

Extension metoda je statická metoda ve statické třídě, kde první parametr má klíčové slovo `this`. Kompilátor ji pak nabídne jako instanční metodu na daném typu.

```csharp
// Definice
public static class StringExtensions
{
    public static bool JePrazdnyNeboNull(this string? s) => string.IsNullOrEmpty(s);
}

// Použití – vypadá jako instanční metoda
"".JePrazdnyNeboNull();    // true
"ahoj".JePrazdnyNeboNull(); // false
```

Extension metody jsou čistě syntaktický cukr – kompilátor je přeloží na volání statické metody. Nemají přístup k privátním členům.

## 2. Příklad

### Extension na vlastní typ

```csharp
public record Objednavka(decimal Castka, bool Zaplacena, DateTime Datum);

public static class ObjednavkaExtensions
{
    // Fluent filtrace
    public static IEnumerable<Objednavka> JenNezaplacene(
        this IEnumerable<Objednavka> objednavky)
        => objednavky.Where(o => !o.Zaplacena);

    public static IEnumerable<Objednavka> ZaRok(
        this IEnumerable<Objednavka> objednavky, int rok)
        => objednavky.Where(o => o.Datum.Year == rok);

    public static decimal CelkovaCastka(
        this IEnumerable<Objednavka> objednavky)
        => objednavky.Sum(o => o.Castka);
}

// Fluent řetězení – čitelné jako věta
decimal dluh = vsechnyObjednavky
    .JenNezaplacene()
    .ZaRok(2024)
    .CelkovaCastka();
```

### Extension na rozhraní

```csharp
// Extension na ILogger – zkratky pro podmíněné logování
public static class LoggerExtensions
{
    public static void LogIfDebug(this ILogger logger, string message)
    {
        if (logger.IsEnabled(LogLevel.Debug))
            logger.LogDebug(message);
    }
}
```

### Extension pro builder pattern

```csharp
// Rozšíření WebApplicationBuilder bez dědičnosti
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddMojeSluzby(
        this IServiceCollection services, IConfiguration config)
    {
        services.AddScoped<IEmailSluzba, EmailSluzba>();
        services.Configure<EmailOptions>(config.GetSection("Email"));
        return services;
    }
}

// V Program.cs:
builder.Services.AddMojeSluzby(builder.Configuration);
```

## 3. Kdy použít

- **Obohacení cizích typů** – přidejte metody na typy z NuGet balíčků nebo BCL, které nemůžete upravit.
- **Fluent API** – řetězitelné operace na kolekcích nebo builder objektech.
- **Helper metody pro rozhraní** – sdílená logika pro všechny implementátory rozhraní bez abstraktní třídy.
- **Organizace kódu** – skupinujte extension metody podle domény do separátních statických tříd.

## 4. Časté chyby

- ❌ **Extension zakrývá instanční metodu** – pokud typ má vlastní metodu se stejným názvem, ta vždy vyhraje. Extension metoda není volána (bez varování!). Ověřte, zda název nekoliduje.
- ❌ **Extension na `object`** – `this object x` se aplikuje na vše; znečišťuje IntelliSense pro každý typ. Buďte co nejkonkrétnější.
- ❌ **Business logika v extension místo doménové vrstvě** – extension metody jsou utility; komplexní doménová pravidla patří do service tříd.
- ❌ **Namespace kolize** – dvě různé extension metody se stejným názvem pro stejný typ z různých namespaců způsobí chybu kompilace. Pojmenovávejte extension metody jednoznačně.

**Další krok:** [Iterátory a yield return](Iterators.md)
