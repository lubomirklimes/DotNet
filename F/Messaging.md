# Messaging (Service Bus, RabbitMQ)

Asynchronní messaging odděluje producenty a konzumenty zpráv, čímž zvyšuje odolnost a škálovatelnost distribuovaných systémů.

## 1. Koncept

| Vlastnost | Azure Service Bus | RabbitMQ |
|-----------|------------------|----------|
| Hosting | Plně spravovaný (PaaS) | Self-hosted nebo CloudAMQP |
| Protokol | AMQP 1.0 | AMQP 0-9-1 |
| Záruky | At-least-once, FIFO sessions | At-least-once, pluginy pro FIFO |
| DLQ | Vestavěná Dead-Letter Queue | Dead-letter exchange |
| Témata/Subscriptions | ✅ | ✅ (exchanges + bindings) |

Základní vzory:
- **Queue** — point-to-point (jeden konzument zpracuje každou zprávu)
- **Topic/Exchange** — publish-subscribe (více konzumentů dostane kopii)
- **Dead-Letter Queue (DLQ)** — zprávy, které se nepodařilo zpracovat

## 2. Příklad

### Azure Service Bus — odesílání

```csharp
// dotnet add package Azure.Messaging.ServiceBus
var client = new ServiceBusClient(
    "mujnamespace.servicebus.windows.net",
    new DefaultAzureCredential());

await using var sender = client.CreateSender("objednavky");
var zprava = new ServiceBusMessage(
    BinaryData.FromObjectAsJson(new { Id = 42, Produkt = "Kniha" }))
{
    ContentType          = "application/json",
    MessageId            = Guid.NewGuid().ToString(),
    TimeToLive           = TimeSpan.FromDays(1)
};
await sender.SendMessageAsync(zprava);
```

### Azure Service Bus — příjem (processor)

```csharp
await using var processor = client.CreateProcessor("objednavky", new ServiceBusProcessorOptions
{
    MaxConcurrentCalls = 4,
    AutoCompleteMessages = false
});

processor.ProcessMessageAsync += async args =>
{
    var data = args.Message.Body.ToObjectFromJson<dynamic>();
    Console.WriteLine($"Objednávka: {data}");
    await args.CompleteMessageAsync(args.Message); // potvrzení zpracování
};
processor.ProcessErrorAsync += args =>
{
    Console.Error.WriteLine(args.Exception);
    return Task.CompletedTask;
};
await processor.StartProcessingAsync();
```

### RabbitMQ s MassTransit (doporučená abstrakce)

```csharp
// dotnet add package MassTransit.RabbitMQ
builder.Services.AddMassTransit(x =>
{
    x.AddConsumer<ObjednavkaConsumer>();
    x.UsingRabbitMq((ctx, cfg) =>
    {
        cfg.Host("rabbitmq://localhost", h =>
        {
            h.Username("guest"); h.Password("guest");
        });
        cfg.ConfigureEndpoints(ctx);
    });
});

public record ObjednavkaVytvorena(int Id, string Produkt);

public class ObjednavkaConsumer : IConsumer<ObjednavkaVytvorena>
{
    public async Task Consume(ConsumeContext<ObjednavkaVytvorena> ctx)
    {
        Console.WriteLine($"Zpracovávám {ctx.Message.Produkt}");
        await Task.CompletedTask;
    }
}
```

## 3. Kdy použít

- **Azure Service Bus** — enterprise aplikace na Azure s požadavkem na DLQ, sessions, transakce, témata.
- **RabbitMQ** — on-premise nebo multi-cloud; flexibilnější routing přes exchange binding.
- **MassTransit** — abstrakce nad oběma brokery; usnadní migraci a přidává saga orchestraci.

## 4. Časté chyby

- ❌ **Bez idempotence konzumenta** — Service Bus garantuje *at-least-once*; konzument musí zvládnout duplicitní zprávy (dedup pomocí `MessageId`).
- ❌ **`AutoCompleteMessages = true` u chybové logiky** — zpráva se odstraní i při výjimce; vždy potvrďte (`Complete`) nebo odmítněte (`DeadLetter`) explicitně.
- ❌ **Příliš velké zprávy** — Service Bus Standard max 256 KB, Premium max 100 MB; pro velká data pošlete referenci na blob.
- ❌ **Bez DLQ monitoringu** — zprávy v DLQ tiše hromadí chyby; nastavte alert na `DeadLetterMessageCount`.

**Další krok:** [Správa tajných klíčů (Key Vault, Managed Identity)](KeyVault.md)
