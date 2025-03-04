# Azure Storage

Azure Storage je spravovaná cloudová úložná služba nabízející čtyři typy úložiště: **Blob** (objekty/soubory), **Queue** (zprávy), **Table** (NoSQL) a **File** (SMB sdílené disky).

## 1. Koncept

| Typ | Použití |
|-----|---------|
| Blob Storage | Soubory, obrázky, zálohy, streamy — kontejnery s bloby |
| Queue Storage | Asynchronní fronty zpráv (max. 64 KB/zpráva, 7 dní TTL) |
| Table Storage | Schéma-free NoSQL tabulky (PartitionKey + RowKey) |
| File Storage | Sdílené disky montovatelné přes SMB pro VM nebo App Service |

Přístup z C# zajišťuje sada balíčků `Azure.Storage.*`. Autentizace preferuje **Managed Identity** přes `DefaultAzureCredential`.

## 2. Příklad

### Blob Storage — upload a download

```csharp
// dotnet add package Azure.Storage.Blobs
using Azure.Storage.Blobs;
using Azure.Identity;

var serviceClient = new BlobServiceClient(
    new Uri("https://mojeulozte.blob.core.windows.net"),
    new DefaultAzureCredential());

var kontejner = serviceClient.GetBlobContainerClient("dokumenty");
await kontejner.CreateIfNotExistsAsync();

// Upload
await using var stream = File.OpenRead("zprava.pdf");
await kontejner.UploadBlobAsync("zprava.pdf", stream);

// Download
var blob = kontejner.GetBlobClient("zprava.pdf");
var response = await blob.DownloadStreamingAsync();
await using var fs = File.Create("zprava_stazena.pdf");
await response.Value.Content.CopyToAsync(fs);

// SAS URL pro dočasný přístup (bez autentizace na klientovi)
var sas = blob.GenerateSasUri(BlobSasPermissions.Read,
    DateTimeOffset.UtcNow.AddHours(1));
```

### Queue Storage — producent a konzument

```csharp
// dotnet add package Azure.Storage.Queues
var queueClient = new QueueClient(
    new Uri("https://mojeulozte.queue.core.windows.net/objednavky"),
    new DefaultAzureCredential());
await queueClient.CreateIfNotExistsAsync();

// Producent
await queueClient.SendMessageAsync(
    Convert.ToBase64String(Encoding.UTF8.GetBytes("""{"id":42}""")));

// Konzument
var messages = await queueClient.ReceiveMessagesAsync(maxMessages: 10);
foreach (var msg in messages.Value)
{
    var json = Encoding.UTF8.GetString(Convert.FromBase64String(msg.MessageText));
    Console.WriteLine(json);
    await queueClient.DeleteMessageAsync(msg.MessageId, msg.PopReceipt);
}
```

### Registrace v DI (doporučený vzor)

```csharp
builder.Services.AddSingleton(new BlobServiceClient(
    new Uri($"https://{builder.Configuration["Storage:Account"]}.blob.core.windows.net"),
    new DefaultAzureCredential()));
```

## 3. Kdy použít

- **Blob**: statické soubory webové aplikace (CDN origin), uživatelské uploady, zálohy.
- **Queue**: oddělení producenta a konzumenta (async zpracování objednávek, emailů). Pro pokročilé scénáře (témata, DLQ, sessions) použijte **Azure Service Bus**.
- **Table**: jednoduchá klíč-hodnota data (logy, telemetrie) kde CosmosDB je předražená.

## 4. Časté chyby

- ❌ **Connection string v kódu nebo appsettings** — vždy použijte `DefaultAzureCredential` + Managed Identity; connection string dejte do Key Vault.
- ❌ **Přímý přístup klienta přes SAS bez expiry** — SAS tokeny bez krátkého TTL se stanou bezpečnostní dírou; generujte je on-demand s expiry max. hodiny.
- ❌ **Velké zprávy ve frontě** — Queue Storage má limit 64 KB; pro větší payloady uložte data do blobu a do fronty pošlete jen referenci.
- ❌ **Bez opakování (retry)** — SDK má vestavěný retry, ale jeho politiku lze přizpůsobit přes `BlobClientOptions.Retry`; nevypínejte jej.

**Další krok:** [Messaging a fronty (Service Bus, RabbitMQ)](Messaging.md)
