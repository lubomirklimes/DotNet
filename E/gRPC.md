# gRPC

gRPC je vysokovýkonný RPC framework od Googlu využívající **Protocol Buffers** (protobuf) pro serializaci a HTTP/2 pro přenos. ASP.NET Core poskytuje `Grpc.AspNetCore` pro server a `Grpc.Net.Client` pro klienta.

## 1. Koncept

Kontrakt API je definován v `.proto` souboru — typovaně, jazykově neutrálně. Z `.proto` souboru jsou při buildu automaticky generovány C# třídy a base-třídy služeb.

Typy komunikace:
| Typ | Popis |
|-----|-------|
| Unary | Jeden request → jedna odpověď (obdoba REST) |
| Server streaming | Jeden request → proud odpovědí |
| Client streaming | Proud requestů → jedna odpověď |
| Bidirectional streaming | Proud requestů ↔ proud odpovědí |

Výhody oproti REST: binární formát (~5× menší než JSON), silné typy, generovaný klient, podpora streamingu.

## 2. Příklad

### Proto soubor

```protobuf
// Protos/objednavky.proto
syntax = "proto3";
option csharp_namespace = "MojeApp.Grpc";

service ObjednavkyService {
  rpc GetObjednavka (ObjednavkaRequest) returns (ObjednavkaResponse);
  rpc StreamObjednavky (StreamRequest) returns (stream ObjednavkaResponse);
}

message ObjednavkaRequest { int32 id = 1; }
message StreamRequest     { int32 uzivatel_id = 1; }
message ObjednavkaResponse {
  int32  id     = 1;
  string popis  = 2;
  double castka = 3;
}
```

### Server (ASP.NET Core)

```csharp
// dotnet add package Grpc.AspNetCore
// Services/ObjednavkyGrpcService.cs
public class ObjednavkyGrpcService(IObjednavkyRepo repo)
    : ObjednavkyService.ObjednavkyServiceBase
{
    public override async Task<ObjednavkaResponse> GetObjednavka(
        ObjednavkaRequest request, ServerCallContext ctx)
    {
        var o = await repo.NactiAsync(request.Id, ctx.CancellationToken);
        return new ObjednavkaResponse { Id = o.Id, Popis = o.Popis, Castka = (double)o.Castka };
    }

    public override async Task StreamObjednavky(
        StreamRequest request,
        IServerStreamWriter<ObjednavkaResponse> stream,
        ServerCallContext ctx)
    {
        await foreach (var o in repo.StreamAsync(request.UzivatelId, ctx.CancellationToken))
        {
            await stream.WriteAsync(
                new ObjednavkaResponse { Id = o.Id, Popis = o.Popis, Castka = (double)o.Castka });
        }
    }
}

// Program.cs
builder.Services.AddGrpc();
app.MapGrpcService<ObjednavkyGrpcService>();
```

### Klient

```csharp
// dotnet add package Grpc.Net.Client
var channel = GrpcChannel.ForAddress("https://localhost:7001");
var client  = new ObjednavkyService.ObjednavkyServiceClient(channel);

// Unary
var odpoved = await client.GetObjednavkaAsync(new ObjednavkaRequest { Id = 42 });

// Server streaming
using var stream = client.StreamObjednavky(new StreamRequest { UzivatelId = 1 });
await foreach (var o in stream.ResponseStream.ReadAllAsync())
    Console.WriteLine(o.Popis);
```

## 3. Kdy použít

- **Interní microservices komunikace** — vysoký throughput, malá latence, silné typy.
- **Streaming** — live data feeds, progressive výsledky AI inference.
- **Polyglot prostředí** — gRPC klient existuje pro Go, Python, Java, Rust aj.

Nepoužívejte gRPC pro veřejná browser API — prohlížeče vyžadují **gRPC-Web** proxy (`Grpc.AspNetCore.Web`) nebo použijte REST/OpenAPI.

## 4. Časté chyby

- ❌ **HTTP/2 není povoleno** — konfigurujte Kestrel s `Protocols = HttpProtocols.Http2`; za IIS/nginx ověřte podporu HTTP/2.
- ❌ **Sdílení `.proto` souborů** — nejpohodlnější je publikovat je jako NuGet balíček nebo Git submodule, aby klient i server měly vždy stejný kontrakt.
- ❌ **Ignorování `ServerCallContext.CancellationToken`** — předávejte token do každého downstream volání; zrušení ze strany klienta jinak nemá efekt.
- ❌ **gRPC přes HTTP (bez TLS) v produkci** — HTTP/2 cleartext funguje interně v Kubernetes, ale bez TLS je provoz nešifrovaný.

**Další krok:** [Docker a Kubernetes v .NET](../F/Docker_Kubernetes.md)
