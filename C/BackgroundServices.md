# Background services a hosted services

`IHostedService` a `BackgroundService` umožňují spouštět **dlouhodobě běžící úlohy** paralelně s hlavní aplikací – zpracování front, pravidelné úlohy (cron), inicializace, cache warmup.

## 1. Koncept

`IHostedService` je rozhraní s metodami `StartAsync` a `StopAsync`. .NET Generic Host je spustí po startu aplikace a zastaví při shutdown.

`BackgroundService` je abstraktní třída implementující `IHostedService` s jedinou metodou `ExecuteAsync(CancellationToken)` – smyčka, která běží po celou dobu životnosti aplikace.

| Typ | Vhodné pro |
|---|---|
| `IHostedService` | Přesná kontrola start/stop (migrace DB, warmup) |
| `BackgroundService` | Kontinuální smyčka (queue worker, scheduler) |

## 2. Příklad

### BackgroundService – worker smyčka

```csharp
public class FrontaWorker(
    IServiceScopeFactory scopeFactory,
    ILogger<FrontaWorker> log) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        log.LogInformation("Worker spuštěn");

        while (!ct.IsCancellationRequested)
        {
            try
            {
                // Scoped service v singleton workeru – přes scope factory
                await using var scope = scopeFactory.CreateAsyncScope();
                var fronta = scope.ServiceProvider.GetRequiredService<IFrontaSluzba>();

                var zprava = await fronta.PrectiAsync(ct);
                if (zprava is not null)
                    await ZpracujAsync(zprava, ct);
                else
                    await Task.Delay(TimeSpan.FromSeconds(1), ct); // backoff
            }
            catch (OperationCanceledException) { break; } // graceful shutdown
            catch (Exception ex)
            {
                log.LogError(ex, "Chyba při zpracování");
                await Task.Delay(TimeSpan.FromSeconds(5), ct); // retry delay
            }
        }
    }
}
```

### IHostedService – jednorázová inicializace

```csharp
public class DatabaseMigrator(IServiceScopeFactory scopeFactory) : IHostedService
{
    public async Task StartAsync(CancellationToken ct)
    {
        await using var scope = scopeFactory.CreateAsyncScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        await db.Database.MigrateAsync(ct);
    }

    public Task StopAsync(CancellationToken ct) => Task.CompletedTask;
}
```

### Registrace v Program.cs

```csharp
builder.Services.AddHostedService<FrontaWorker>();
builder.Services.AddHostedService<DatabaseMigrator>();

// Pro Worker Service projekt (bez HTTP):
// dotnet new worker -n MujWorker
```

### Pravidelné spouštění (scheduler)

```csharp
public class DenníReport(IReportSluzba sluzba, ILogger<DenníReport> log)
    : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        // Zarovnání na celou hodinu při startu
        var prvniSpusteni = DateTime.Now.Date.AddDays(1).AddHours(6);
        await Task.Delay(prvniSpusteni - DateTime.Now, ct);

        using var timer = new PeriodicTimer(TimeSpan.FromHours(24));
        while (await timer.WaitForNextTickAsync(ct))
        {
            try   { await sluzba.VygenerujAsync(ct); }
            catch (Exception ex) { log.LogError(ex, "Report selhal"); }
        }
    }
}
```

## 3. Kdy použít

- **Queue worker** – odběr zpráv z Azure Service Bus, RabbitMQ, Channel nebo DB fronty.
- **Cache warmup** – předplnění cache při startu před prvním requestem.
- **DB migrace** – spuštění `MigrateAsync` před otevřením HTTP endpointů.
- **Schedulované úlohy** – `PeriodicTimer` místo externího scheduleru pro jednoduché případy.

## 4. Časté chyby

- ❌ **Přímé použití Scoped služby v singleton workeru** – `BackgroundService` je singleton; přímá injekce `DbContext` (scoped) způsobí výjimku nebo neočekávané chování. Vždy `IServiceScopeFactory`.
- ❌ **Ignorování `CancellationToken`** – aplikace se při shutdown zasekne a čeká na timeout (výchozí 5s), pokud worker nezareaguje na zrušení.
- ❌ **`ExecuteAsync` bez try/catch** – nezachycená výjimka zastaví workera; aplikace běží dál bez workera a bez upozornění.
- ❌ **`Thread.Sleep` místo `await Task.Delay`** – blokuje vlákno ThreadPoolu; aplikace spotřebovává vlákno i při nečinnosti.

**Další krok:** [Windows Forms (WinForms)](../D/WinForms.md)
