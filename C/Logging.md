# Logování a diagnostika

Logování zaznamenává události v aplikaci – je nezbytné pro ladění a monitorování.

## Vestavěné logování `ILogger` (.NET)

```csharp
using Microsoft.Extensions.Logging;

// Získaní loggeru přes DI (viz sekci DependencyInjection)
ILogger<MojeTrida> logger = ...;

logger.LogInformation("Aplikace spusštěna.");
logger.LogWarning("Nízá kapacita: {Kapacita} MB", kapacita);
logger.LogError(ex, "Chyba při zpracování požadavku {Id}", requestId);
```

Úrovně logování (od nejméně po nejvíce závažné):
`Trace` < `Debug` < `Information` < `Warning` < `Error` < `Critical`

## Konfigurace v `Program.cs`

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Logging.AddConsole();
builder.Logging.SetMinimumLevel(LogLevel.Information);
```

## Serilog – oblíbená externí knihovna

```csharp
// NuGet: Serilog.AspNetCore
Log.Logger = new LoggerConfiguration()
    .WriteTo.Console()
    .WriteTo.File("log.txt", rollingInterval: RollingInterval.Day)
    .CreateLogger();

Log.Information("Start aplikace");
Log.Error(ex, "Chyba!");
```

## `Debug` a `Trace` pro vývoj

```csharp
using System.Diagnostics;

Debug.WriteLine("Toto se zobrazí jen v DEBUG režimu.");
Trace.TraceInformation("Trasovací zpráva.");
```

---

<details>
<summary>Volitelné: Strukturované logování a OpenTelemetry</summary>

Strukturované logování ukládá logy jako JSON – lehče se prohledávají v nástrojích jako Seq nebo Elastic Stack:

```csharp
logger.LogInformation("Objednávka {OrderId} od {UserId} zpracována.", orderId, userId);
```

OpenTelemetry je moderní standard pro distribuované trasování – viz [opentelemetry.io](https://opentelemetry.io/).

</details>

---

**Další krok:** [Základy práce se sítí](Networking.md)
