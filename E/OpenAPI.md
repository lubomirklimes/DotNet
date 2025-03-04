# OpenAPI (Swagger)

OpenAPI je standard pro popis REST API ve strojově čitelném formátu. ASP.NET Core jej podporuje nativně od verze 9; pro starší verze se používá **Swashbuckle** nebo **NSwag**.

## 1. Koncept

OpenAPI dokument (JSON/YAML) popisuje endpointy, parametry, těla požadavků, odpovědi a schémata. Z dokumentu lze automaticky generovat klientský kód, testovací UI (Swagger UI, Scalar) nebo API testy.

ASP.NET Core 9+ obsahuje balíček `Microsoft.AspNetCore.OpenApi` přímo v SDK. Pro starší verze:
- `Swashbuckle.AspNetCore` — nejrozšířenější, integrované Swagger UI
- `NSwag.AspNetCore` — generování klientů pro C# a TypeScript

Metadata endpointů se zadávají přes atributy (`[Tags]`, `[EndpointSummary]`, `[ProducesResponseType]`) nebo fluent API (`.WithName()`, `.WithSummary()`, `.Produces<T>()`).

## 2. Příklad

### ASP.NET Core 9 — nativní OpenAPI

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddOpenApi(); // Microsoft.AspNetCore.OpenApi

var app = builder.Build();

if (app.Environment.IsDevelopment())
    app.MapOpenApi(); // vystaví /openapi/v1.json

app.MapGet("/produkty/{id}", async (int id, IProduktyRepo repo, CancellationToken ct) =>
{
    var produkt = await repo.NactiAsync(id, ct);
    return produkt is null ? Results.NotFound() : Results.Ok(produkt);
})
.WithName("GetProdukt")
.WithSummary("Načte produkt podle ID")
.WithTags("Produkty")
.Produces<Produkt>()
.Produces(404);

app.Run();
```

### Swashbuckle (ASP.NET Core 6–8)

```csharp
// dotnet add package Swashbuckle.AspNetCore
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(o =>
{
    o.SwaggerDoc("v1", new() { Title = "Moje API", Version = "v1" });
    // zahrnutí XML komentářů:
    var xml = Path.Combine(AppContext.BaseDirectory, "MojeApi.xml");
    o.IncludeXmlComments(xml);
});

app.UseSwagger();
app.UseSwaggerUI(o => o.SwaggerEndpoint("/swagger/v1/swagger.json", "Moje API v1"));
```

### Atributy pro Controllers

```csharp
/// <summary>Vrátí seznam produktů.</summary>
/// <response code="200">OK — seznam produktů</response>
/// <response code="401">Neautorizováno</response>
[HttpGet]
[ProducesResponseType(typeof(IEnumerable<Produkt>), 200)]
[ProducesResponseType(401)]
public async Task<IActionResult> GetVsechny(CancellationToken ct)
    => Ok(await _repo.VsechnyAsync(ct));
```

## 3. Kdy použít

- **Vždy u veřejných nebo interních REST API** — dokumentace, testování, generování klientů.
- **Scalar jako UI v .NET 9** — modernější alternativa Swagger UI, stačí přidat `app.MapScalarApiReference()`.
- **NSwag** — pokud potřebujete automaticky generovat C# nebo TypeScript klienta z OpenAPI dokumentu.

## 4. Časté chyby

- ❌ **`AddEndpointsApiExplorer` chybí** — Swashbuckle nevidí Minimal API endpointy bez tohoto volání.
- ❌ **Chybějící `.Produces<T>()`** — bez anotace Swagger zobrazuje jen `200 string`; vždy specifikujte návratový typ.
- ❌ **Vystavení v produkci** — `/swagger` nebo `/openapi` dostupné veřejně odhaluje interní strukturu API; podmíněte `IsDevelopment()` nebo chraňte autentizací.
- ❌ **Duplicitní operace ID** — více endpointů bez `.WithName()` způsobí konflikty při generování klientů.

**Další krok:** [SignalR – real-time komunikace](SignalR.md)
