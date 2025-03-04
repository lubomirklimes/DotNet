# Dependency Injection (DI) a práce se službami

Dependency Injection je vzor, který **zajišťuje závislosti** (služby, klienty, ...) objektu zvenku místo toho, aby si je vytvářel sám.

## Proč DI?

```csharp
// Bez DI - třída si vytváří závislost sama (těžkou\ testání)
class ObjednavkoveSluzby
{
    private readonly EmailSluzba _email = new EmailSluzba(); // pevná vazba
}

// S DI - závislost přichází zvenku (snadné testání, zaměnitelné)
class ObjednavkoveSluzby(IEmailSluzba emailSluzba)
{
    // emailSluzba předá DI kontejner
}
```

## Registrace služeb

```csharp
var builder = WebApplication.CreateBuilder(args);

// Registrace služeb do DI kontejneru
builder.Services.AddSingleton<IKonfigurace, Konfigurace>();   // jedna instance pro celý život app
builder.Services.AddScoped<IObjednavkyRepo, ObjednavkyRepo>(); // nová instance per request
builder.Services.AddTransient<IEmailSluzba, SmtpEmailSluzba>(); // nová instance vždy
```

| Životnost | Kdy použít |
|---|---|
| `Singleton` | Společná konfigurace, cache, obecné služby |
| `Scoped` | Databázové kontexty, služby per HTTP request |
| `Transient` | Lehké, bezstavové služby |

## Získání služby (injekce konstruktorem)

```csharp
class ObjednavkovyController(IObjednavkyRepo repo, IEmailSluzba email)
{
    public async Task<IActionResult> Create(Objednavka o)
    {
        await repo.UlozAsync(o);
        await email.OdeslatsAsync(o.Email, "Potvrzení", "Vaše objednávka proběhla.");
        return Ok();
    }
}
```

## Rozhraní + implementace

```csharp
interface IEmailSluzba
{
    Task OdeslatAsync(string adresat, string predmet, string telo);
}

class SmtpEmailSluzba : IEmailSluzba
{
    public async Task OdeslatAsync(string adresat, string predmet, string telo)
    {
        // Implementace odeslání e-mailu přes SMTP
    }
}
```

---

<details>
<summary>Volitelné: Ruční přístup k službám a options vzor</summary>

```csharp
// Ruční přístup (ServiceLocator - obecně se nedoporučuje)
var sluzba = app.Services.GetRequiredService<IEmailSluzba>();

// Options vzor pro konfigurovanie služeb
builder.Services.Configure<SmtpNastaveni>(
    builder.Configuration.GetSection("Smtp"));

// Použití v službě
class SmtpEmailSluzba(IOptions<SmtpNastaveni> options)
{
    private readonly SmtpNastaveni _nastaveni = options.Value;
}
```

</details>

---

**Další krok:** [Konfigurace aplikace (appsettings, IOptions)](Configuration.md)
