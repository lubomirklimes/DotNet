# Základy práce se sítí

## HTTP požadavky s `HttpClient`

```csharp
using System.Net.Http;
using System.Net.Http.Json;

HttpClient client = new HttpClient(); // pouze pro ukázku; v ASP.NET Core použijte IHttpClientFactory

// GET
string obsah = await client.GetStringAsync("https://api.example.com/data");

// GET s deserializací JSON
var produkt = await client.GetFromJsonAsync<Produkt>("https://api.example.com/products/1");

// POST
var novy = new Produkt { Nazev = "Test", Cena = 99 };
var response = await client.PostAsJsonAsync("https://api.example.com/products", novy);
response.EnsureSuccessStatusCode();
```

## Registrace `HttpClient` přes DI (doporučený přístup)

```csharp
// Program.cs
builder.Services.AddHttpClient<ProduktService>(client =>
{
    client.BaseAddress = new Uri("https://api.example.com/");
    client.Timeout = TimeSpan.FromSeconds(30);
});
```

```csharp
// ProduktService.cs
class ProduktService(HttpClient client)
{
    public Task<Produkt?> GetProduktAsync(int id) =>
        client.GetFromJsonAsync<Produkt>($"products/{id}");
}
```

## Ošetření chyb

```csharp
try
{
    var data = await client.GetStringAsync("https://api.example.com/data");
}
catch (HttpRequestException ex)
{
    Console.WriteLine($"HTTP chyba: {ex.StatusCode} – {ex.Message}");
}
catch (TaskCanceledException)
{
    Console.WriteLine("Požadavek vypršel (timeout).");
}
```

---

<details>
<summary>Volitelné: WebSockets a gRPC</summary>

Pro **obousměrnou komunikaci** v reálném čase použij WebSockets:

```csharp
using var ws = new ClientWebSocket();
await ws.ConnectAsync(new Uri("wss://example.com/ws"), CancellationToken.None);
```

Pro **výkonné API** mězi službami se hodí gRPC (viz [grpc.io](https://grpc.io/)).

</details>

---

**Další krok:** [Serializace a deserializace](Serialization.md)
