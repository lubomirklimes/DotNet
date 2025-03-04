# Minimal APIs

Minimal API je nejjednodušší způsob, jak vytvořit HTTP API v .NET – bez Controllerů, s minmálním kódem.

## Vytvoření projektu

```
dotnet new web -n MojeApi
```

## Základní API

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();
app.UseSwagger();
app.UseSwaggerUI();

// Endpointy
app.MapGet("/produkty", () => new[] { "Chléb", "Mléko" });

app.MapGet("/produkty/{id:int}", (int id) =>
    id == 1 ? Results.Ok(new { Id = 1, Nazev = "Chléb" })
            : Results.NotFound());

app.MapPost("/produkty", (Produkt p) =>
{
    // uložit do DB
    return Results.Created($"/produkty/{p.Id}", p);
});

app.Run();
```

## S DI a službami

```csharp
builder.Services.AddScoped<IProduktService, ProduktService>();

app.MapGet("/produkty", async (IProduktService service) =>
    await service.GetVsechnyAsync());

app.MapGet("/produkty/{id}", async (int id, IProduktService service) =>
{
    var p = await service.GetByIdAsync(id);
    return p is null ? Results.NotFound() : Results.Ok(p);
});
```

## Skupiny endpointů a validace

```csharp
var produkty = app.MapGroup("/produkty").WithTags("Produkty");

produkty.MapGet("/", GetAll);
produkty.MapGet("/{id}", GetById);
produkty.MapPost("/", Create).WithParameterValidation();
```

## Autorizace

```csharp
builder.Services.AddAuthentication().AddJwtBearer();
builder.Services.AddAuthorization();

app.MapGet("/tajne", () => "Tajná data")
   .RequireAuthorization();
```

---

<details>
<summary>Volitelné: OpenAPI, Rate Limiting a versioning</summary>

**.NET 9 OpenAPI** (bez Swashbuckle):

```csharp
builder.Services.AddOpenApi();
app.MapOpenApi();
```

**Rate limiting:**

```csharp
builder.Services.AddRateLimiter(o =>
    o.AddFixedWindowLimiter("fixed", opt => {
        opt.PermitLimit = 10;
        opt.Window = TimeSpan.FromSeconds(1);
    }));

app.MapGet("/api", () => "OK").RequireRateLimiting("fixed");
```

</details>

---

**Další krok:** [REST API – konvence, HttpClient, verzování](REST_API.md)
