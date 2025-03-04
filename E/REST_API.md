# REST API v .NET – konvence, HttpClient a verzování

REST API je nejrozšířenější způsob komunikace mezi službami a klienty. .NET nabízí jak serverovou stranu (ASP.NET Core), tak klientskou (HttpClient s IHttpClientFactory).

## 1. Koncept

### REST konvence

| HTTP metoda | Akce | Příklad URL |
|---|---|---|
| GET | Čtení | `/api/produkty`, `/api/produkty/42` |
| POST | Vytvoření | `/api/produkty` |
| PUT | Úplná náhrada | `/api/produkty/42` |
| PATCH | Částečná úprava | `/api/produkty/42` |
| DELETE | Smazání | `/api/produkty/42` |

### HTTP stavové kódy

| Kód | Situace |
|---|---|
| 200 OK | Úspěšné GET / PUT |
| 201 Created | Úspěšné POST (+ Location header) |
| 204 No Content | Úspěšné DELETE |
| 400 Bad Request | Nevalidní vstup |
| 401 Unauthorized | Chybí autentizace |
| 403 Forbidden | Nemáte právo |
| 404 Not Found | Zdroj neexistuje |
| 409 Conflict | Konflikt (duplicate key) |
| 500 Internal Server Error | Neočekávaná chyba serveru |

## 2. Příklad

### Minimal API REST endpoint

```csharp
var app = builder.Build();

app.MapGet("/api/produkty",
    async (IProduktService svc, CancellationToken ct) =>
        Results.Ok(await svc.GetAllAsync(ct)));

app.MapGet("/api/produkty/{id:int}",
    async (int id, IProduktService svc, CancellationToken ct) =>
        await svc.GetByIdAsync(id, ct) is { } p ? Results.Ok(p) : Results.NotFound());

app.MapPost("/api/produkty",
    async (ProduktDto dto, IProduktService svc, CancellationToken ct) =>
    {
        var created = await svc.CreateAsync(dto, ct);
        return Results.CreatedAtRoute("GetProdukt", new { id = created.Id }, created);
    });

app.MapDelete("/api/produkty/{id:int}",
    async (int id, IProduktService svc, CancellationToken ct) =>
    {
        await svc.DeleteAsync(id, ct);
        return Results.NoContent();
    });
```

### HttpClient s IHttpClientFactory (klient)

```csharp
// Registrace v Program.cs
builder.Services.AddHttpClient<ProduktApiKlient>(client =>
{
    client.BaseAddress = new Uri("https://api.example.com/");
    client.DefaultRequestHeaders.Add("Accept", "application/json");
});

// Typed klient
public class ProduktApiKlient(HttpClient http)
{
    public async Task<List<Produkt>> GetAllAsync(CancellationToken ct)
    {
        var response = await http.GetAsync("/api/produkty", ct);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<List<Produkt>>(ct) ?? [];
    }
}
```

### API verzování

```csharp
// Přes URL segment – nejčitelnější
app.MapGroup("/api/v1/produkty").MapProduktEndpointsV1();
app.MapGroup("/api/v2/produkty").MapProduktEndpointsV2();

// Nebo přes Asp.Versioning NuGet balíček pro Header/QueryString verzování
```

## 3. Kdy použít

- **Typed HttpClient** místo `new HttpClient()` – `IHttpClientFactory` spravuje životnost, pooling a `HttpMessageHandler`.
- **`Results.CreatedAtRoute`** po POST – klient dostane URL nově vytvořeného zdroje v `Location` hlavičce.
- **Problem Details (RFC 7807)** pro chybové odpovědi – `builder.Services.AddProblemDetails()` zajistí standardní formát chybových odpovědí.

## 4. Časté chyby

- ❌ **`new HttpClient()` v každém requestu** – `HttpClient` není určen k opakovanému vytváření; vede k vyčerpání socketů. Vždy `IHttpClientFactory`.
- ❌ **Neověřený `EnsureSuccessStatusCode`** – vyhodí `HttpRequestException` bez těla odpovědi; pro detaily použijte `response.Content.ReadAsStringAsync()` při chybě.
- ❌ **Vracení 200 pro POST** – po vytvoření entity vraťte 201 Created s `Location` headerem.
- ❌ **Ukládání `HttpClient` jako singleton ručně** – `HttpClient` cachuje DNS; v long-lived singleton se DNS změny neprojeví. `IHttpClientFactory` to řeší rotací handlerů.

**Další krok:** [Middleware a pipeline v ASP.NET Core](Middleware.md)
