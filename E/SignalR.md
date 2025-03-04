# SignalR

ASP.NET Core SignalR je knihovna pro **real-time obousměrnou komunikaci** mezi serverem a klienty. Interně používá WebSocket (s fallbackem na Server-Sent Events nebo Long Polling).

## 1. Koncept

Jádrem SignalR je **Hub** — třída, jejíž metody mohou klienti volat ze JavaScriptu/C# a server může naopak volat metody registrované na klientech. Připojení jsou udržována po celou dobu session.

Skupiny (`Groups`) umožňují rozesílat zprávy podmnožině klientů (např. místnost chatu, tenant v multi-tenant aplikaci).

Škálování na více instancí vyžaduje **backplane** (Azure SignalR Service nebo Redis).

## 2. Příklad

### Hub

```csharp
// Hubs/ChatHub.cs
public class ChatHub : Hub
{
    // Klient volá: connection.invoke("OdesliZpravu", "pokoj1", "Ahoj!")
    public async Task OdesliZpravu(string skupina, string zprava)
    {
        // Rozešle všem klientům ve skupině
        await Clients.Group(skupina).SendAsync("PrijmiZpravu",
            Context.UserIdentifier, zprava);
    }

    public async Task PridejSeDoSkupiny(string skupina)
        => await Groups.AddToGroupAsync(Context.ConnectionId, skupina);

    public override async Task OnDisconnectedAsync(Exception? ex)
    {
        // Cleanup při odpojení
        await base.OnDisconnectedAsync(ex);
    }
}
```

### Registrace

```csharp
// Program.cs
builder.Services.AddSignalR();

app.MapHub<ChatHub>("/hubs/chat");
```

### Volání klienta ze serveru (bez připojení klienta)

```csharp
// Inject IHubContext<ChatHub> do service/controller
public class OznameniService(IHubContext<ChatHub> hub)
{
    public async Task RozesliAsync(string skupina, string text, CancellationToken ct)
        => await hub.Clients.Group(skupina).SendAsync("PrijmiZpravu", "server", text, ct);
}
```

### JavaScript klient

```javascript
const connection = new signalR.HubConnectionBuilder()
    .withUrl("/hubs/chat")
    .withAutomaticReconnect()
    .build();

connection.on("PrijmiZpravu", (odesilatel, zprava) =>
    console.log(`${odesilatel}: ${zprava}`));

await connection.start();
await connection.invoke("PridejSeDoSkupiny", "pokoj1");
await connection.invoke("OdesliZpravu", "pokoj1", "Ahoj!");
```

### Azure SignalR Service (škálování)

```csharp
// dotnet add package Microsoft.Azure.SignalR
builder.Services.AddSignalR()
    .AddAzureSignalR(builder.Configuration["Azure:SignalR:ConnectionString"]);
```

## 3. Kdy použít

- **Real-time dashboardy** — aktualizace cen, monitorovací metriky.
- **Kolaborativní aplikace** — chat, sdílené editory, live komentáře.
- **Notifikace** — push upozornění bez pollingu.

Pokud potřebujete pouze jednosměrný push ze serveru (bez volání metod klienta zpět), zvažte **Server-Sent Events** (`IAsyncEnumerable<T>` přes Minimal API) jako jednodušší alternativu.

## 4. Časté chyby

- ❌ **Chybí CORS pro SignalR** — WebSocket handshake potřebuje explicitní `WithOrigins` + `AllowCredentials()` v CORS politice.
- ❌ **Volání `hub.Clients.All` z Hub metody pro broadcast** — správně funguje, ale snadno přetíží server; použijte skupiny.
- ❌ **Ztráta zpráv při reconnectu** — SignalR nezaručuje doručení; pro spolehlivé fronty použijte Service Bus nebo Channels.
- ❌ **Sticky sessions bez backplane** — multi-instance deployment bez Redis/Azure SignalR Service způsobí, že klienti připojení na jiný uzel zprávy nedostanou.

**Další krok:** [gRPC v .NET](gRPC.md)
