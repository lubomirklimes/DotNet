# Konfigurace aplikace v .NET

.NET poskytuje jednotný systém konfigurace přes `IConfiguration`. Hodnoty mohou pocházet z `appsettings.json`, proměnných prostředí, příkazové řádky, Azure Key Vault nebo vlastních zdrojů – vždy se skládají do jednoho stromu s prioritami.

## 1. Koncept

### Hierarchie zdrojů (výchozí pořadí v ASP.NET Core)

Vyšší číslo = vyšší priorita (přepisuje nižší):

1. `appsettings.json`
2. `appsettings.{Environment}.json` (Development, Production)
3. User Secrets (jen Development)
4. Proměnné prostředí
5. Argumenty příkazové řádky

### Options pattern

`IOptions<T>`, `IOptionsSnapshot<T>` a `IOptionsMonitor<T>` mapují sekci konfigurace na typovanou třídu.

| Typ | Životnost | Reload za běhu |
|---|---|---|
| `IOptions<T>` | Singleton | Ne |
| `IOptionsSnapshot<T>` | Scoped | Ano (per-request) |
| `IOptionsMonitor<T>` | Singleton | Ano (okamžitý) |

## 2. Příklad

### appsettings.json

```json
{
  "Databaze": {
    "ConnectionString": "Server=localhost;Database=Moje",
    "MaxConnections": 20
  },
  "Email": {
    "Server": "smtp.example.com",
    "Port": 587
  }
}
```

### Čtení přes IConfiguration přímo

```csharp
// V Program.cs nebo konstruktoru přes DI
public class Sluzba(IConfiguration config)
{
    public void Spust()
    {
        string? cs   = config["Databaze:ConnectionString"];
        int max      = config.GetValue<int>("Databaze:MaxConnections", defaultValue: 10);
        var emailSek = config.GetSection("Email");
        string? server = emailSek["Server"];
    }
}
```

### Options pattern – typovaná konfigurace

```csharp
// Třída možností
public class EmailOptions
{
    public const string SekceNazev = "Email";
    public string Server { get; set; } = "";
    public int Port { get; set; } = 587;
}

// Registrace v Program.cs
builder.Services
    .AddOptions<EmailOptions>()
    .BindConfiguration(EmailOptions.SekceNazev)
    .ValidateDataAnnotations()  // validace přes [Required], [Range] atd.
    .ValidateOnStart();         // chyba při startu, ne až při prvním použití

// Použití v třídě
public class EmailSluzba(IOptions<EmailOptions> options)
{
    private readonly EmailOptions _opts = options.Value;

    public Task OdesliAsync(string komu, string text)
    {
        // _opts.Server, _opts.Port jsou bezpečně typované
    }
}
```

### Proměnné prostředí a Docker

```bash
# Dvojtečka v klíči -> double underscore v env var
Databaze__ConnectionString=Server=proddb;Database=Moje
Databaze__MaxConnections=50
```

```yaml
# docker-compose.yml
environment:
  - Databaze__ConnectionString=Server=db;Database=Moje
  - ASPNETCORE_ENVIRONMENT=Production
```

## 3. Kdy použít

- **Options pattern vždy** místo přímého `IConfiguration["klic"]` v service třídách; typové bezpečí + testovatelnost.
- **`ValidateOnStart()`** pro kritické hodnoty (connection stringy, API klíče) – havárie při startu je lepší než za běhu.
- **`IOptionsMonitor<T>`** pro konfiguraci, která se mění za běhu (feature flags, limity).
- **User Secrets** pro lokální vývoj – nikdy neukládejte hesla do `appsettings.json` v repozitáři.

## 4. Časté chyby

- ❌ **Čtení `IConfiguration` přímo v service třídách** – testování vyžaduje ruční sestavení `IConfiguration`; Options pattern je testovatelný přes `Options.Create(new MyOptions {...})`.
- ❌ **Chybějící `ValidateOnStart`** – nevalidovaná konfigurace selže až při prvním použití (první request), ne při startu aplikace.
- ❌ **Tajné hodnoty v appsettings.json commitnutém do gitu** – používejte User Secrets lokálně, proměnné prostředí v CI/CD, Azure Key Vault v produkci.
- ❌ **`IOptions<T>` kde je potřeba reload** – `IOptions<T>` cachuje hodnotu při startu; pro dynamické hodnoty použijte `IOptionsMonitor<T>`.

**Další krok:** [Background services a hosted services](BackgroundServices.md)
