# Observability (Logging, Metrics, Tracing)

Observability je schopnost porozumět vnitřnímu stavu systému z jeho vnějších výstupů. Tři pilíře: **strukturované logy**, **metriky** a **distribuované trasování**.

## 1. Koncept

| Pilíř | Co měří | Nástroj v .NET |
|-------|---------|----------------|
| Logy | Diskrétní události (chyby, požadavky) | `ILogger<T>`, Serilog |
| Metriky | Agregovaná čísla (RPS, latence P99, velikost fronty) | `System.Diagnostics.Metrics`, Prometheus |
| Tracing | Průchod requestu přes služby (spans, trace ID) | `System.Diagnostics.Activity`, OpenTelemetry |

**OpenTelemetry** (OTel) je vendor-neutrální standard. Sada balíčků `OpenTelemetry.*` exportuje data do Jaeger, Zipkin, Azure Monitor, Grafana aj.

## 2. Příklad

### Strukturované logování se Serilog

```csharp
// dotnet add package Serilog.AspNetCore Serilog.Sinks.Console Serilog.Sinks.Seq
builder.Host.UseSerilog((ctx, lc) => lc
    .ReadFrom.Configuration(ctx.Configuration)
    .Enrich.FromLogContext()
    .WriteTo.Console(outputTemplate:
        "[{Timestamp:HH:mm:ss} {Level:u3}] {SourceContext}: {Message:lj}{NewLine}{Exception}")
    .WriteTo.Seq("http://localhost:5341"));

// Použití v service
public class ObjednavkyService(ILogger<ObjednavkyService> logger)
{
    public async Task ZpracujAsync(int id, CancellationToken ct)
    {
        logger.LogInformation("Zpracovávám objednávku {ObjednavkaId}", id);
        // ...
    }
}
```

### OpenTelemetry — logy + metriky + tracing

```csharp
// dotnet add package OpenTelemetry.Extensions.Hosting
// dotnet add package OpenTelemetry.Instrumentation.AspNetCore
// dotnet add package OpenTelemetry.Exporter.Console (nebo OTLP)
builder.Services.AddOpenTelemetry()
    .WithTracing(b => b
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddConsoleExporter())
    .WithMetrics(b => b
        .AddAspNetCoreInstrumentation()
        .AddRuntimeInstrumentation()
        .AddConsoleExporter());
```

### Vlastní metrika

```csharp
using System.Diagnostics.Metrics;

public class ObjednavkyMetriky
{
    private static readonly Meter _meter = new("MojeApp.Objednavky");
    private static readonly Counter<long> _pocet =
        _meter.CreateCounter<long>("objednavky.celkem", "ks", "Celkový počet objednávek");
    private static readonly Histogram<double> _cas =
        _meter.CreateHistogram<double>("objednavky.zpracovani_ms", "ms", "Čas zpracování");

    public void ZaznamObjednavka() => _pocet.Add(1);
    public void ZaznamCas(double ms) => _cas.Record(ms);
}
```

### Vlastní Activity (span)

```csharp
using System.Diagnostics;

private static readonly ActivitySource _source = new("MojeApp");

public async Task ZpracujAsync(int id, CancellationToken ct)
{
    using var activity = _source.StartActivity("ZpracujObjednavku");
    activity?.SetTag("objednavka.id", id);
    // ...
}
```

## 3. Kdy použít

- **Vždy** — každá produkční aplikace potřebuje alespoň strukturované logy a základní metriky.
- **OpenTelemetry** jako standard — umožňuje přepnutí backendu (Grafana, Azure Monitor, Datadog) bez změny kódu.
- **Azure Monitor / Application Insights** — pro aplikace na Azure; OTel exporter: `Azure.Monitor.OpenTelemetry.AspNetCore`.

## 4. Časté chyby

- ❌ **`Console.WriteLine` místo `ILogger`** — konzolový výpis není strukturovaný, nezachycuje úroveň ani kontext.
- ❌ **Logování citlivých dat** — hesla, tokeny, PII nesmí vstoupit do logů; maskujte je před logováním.
- ❌ **Chybějící korelační ID** — bez `TraceId` je nemožné sledovat request napříč mikroservicemi; OTel to řeší automaticky, ale jen pokud je instrumentace aktivní na celé cestě.
- ❌ **Log level Information v produkci** — příliš detailní logování zvyšuje náklady i šum; nastavte `Warning` s možností dynamické změny přes `IOptionsMonitor<LoggerFilterOptions>`.

**Další krok:** [Unity a C# – herní engine](../G/Unity.md)
