# Minimalizace doby startu aplikace a warm-up

Startup time je čas od spuštění procesu do momentu, kdy aplikace dokáže obsloužit první request. V Kubernetes, serverless nebo mikroservisní architektuře je pomalý start přímý problém: ovlivňuje latenci při scale-out, prodražuje cold starty a komplikuje rolling deploymenty.

## 1. Koncept

### Co zabírá čas při startu .NET aplikace

1. **Načítání assembly** – CLR musí načíst a ověřit IL kód pro každou knihovnu.
2. **JIT kompilace** – metody se kompilují do nativního kódu při prvním volání.
3. **Inicializace DI kontejneru** – registrace, validace, vytvoření prvních singleton instancí.
4. **Middleware pipeline** – každý middleware je inicializován a zapojen.
5. **Databázová připojení a warmup** – první dotazy jsou pomalejší (connection pool, query plan cache).

Každá vrstva přidává desítky až stovky milisekund. Pro aplikaci, která startuje jednou za den, to nevadí. Pro serverless funkci, která startuje při každém requestu, je to kritické.

## 2. Příklad

### ReadyToRun – zkrácení JIT startup

```xml
<!-- .csproj -->
<PropertyGroup>
  <PublishReadyToRun>true</PublishReadyToRun>
</PropertyGroup>
```

```bash
dotnet publish -c Release -r linux-x64 --self-contained
```

R2R předkompiluje IL kód do nativního při publishování. JIT se stále použije pro části, které R2R nepokryla (nebo pro Tier 1 re-kompilaci), ale první startup je výrazně rychlejší.

### Trimming – menší assembly = rychlejší načítání

```xml
<PropertyGroup>
  <PublishTrimmed>true</PublishTrimmed>
  <TrimMode>link</TrimMode>
</PropertyGroup>
```

Trimmer odstraní nepoužívaný kód z assembly. Menší binárka = méně kódu k načtení a JIT-kompilaci. **Pozor:** trimming je nekompatibilní s reflexí bez anotací – testujte důkladně.

### Lazy inicializace závislostí

```csharp
// ❌ Všechno inicializováno při startu – i služby, které možná nebudou potřeba
builder.Services.AddSingleton<TěžkáSluzba>(); // konstruktor trvá 500 ms

// ✅ Lazy<T> – inicializace proběhne až při prvním použití
builder.Services.AddSingleton<Lazy<TěžkáSluzba>>(sp =>
    new Lazy<TěžkáSluzba>(() => sp.GetRequiredService<TěžkáSluzba>()));

// Použití
public class MujController(Lazy<TěžkáSluzba> sluzba)
{
    public IActionResult Get()
    {
        // Inicializace proběhne teprve tady, ne při startu
        return Ok(sluzba.Value.ZpracujAsync());
    }
}
```

### IHostedService warm-up před přijetím requestů

```csharp
// Warm-up: zahřej databázi a JIT před tím, než load balancer přesměruje provoz
public class WarmupService(IDbContextFactory<AppDbContext> dbFactory, ILogger<WarmupService> log)
    : IHostedService
{
    public async Task StartAsync(CancellationToken ct)
    {
        log.LogInformation("Zahřívám aplikaci...");

        await using var db = await dbFactory.CreateDbContextAsync(ct);
        // Jednoduchý dotaz zahřeje connection pool a zkompiluje query
        _ = await db.Produkty.Take(1).ToListAsync(ct);

        log.LogInformation("Warm-up dokončen.");
    }

    public Task StopAsync(CancellationToken ct) => Task.CompletedTask;
}

// Registrace – spustí se před přijetím requestů
builder.Services.AddHostedService<WarmupService>();
```

### Health check – load balancer nesmí posílat provoz před warm-upem

```csharp
builder.Services.AddHealthChecks()
    .AddCheck<WarmupHealthCheck>("warmup");

// Health check vrátí Unhealthy, dokud warm-up nedokončí
public class WarmupHealthCheck(WarmupService warmup) : IHealthCheck
{
    public Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext ctx, CancellationToken ct)
        => Task.FromResult(warmup.IsReady
            ? HealthCheckResult.Healthy()
            : HealthCheckResult.Unhealthy("Warmup neprobehlo"));
}
```

## 3. Kdy použít

- **ReadyToRun** – vždy u aplikací, kde záleží na startup time (API, mikroservisy).
- **NativeAOT** – CLI nástroje, serverless funkce s cold startem, kontejnery s paměťovým limitem (viz [JIT_AOT.md](JIT_AOT.md)).
- **Lazy inicializace** – DI služby, které nejsou potřeba při každém requestu.
- **Warm-up IHostedService** – před přijetím provozu zahřejte DB connection pool, JIT hot paths a externí připojení.
- **Trimming** – single-file publish, serverless, IoT.

## 4. Časté chyby

- ❌ **Těžká logika v konstruktorech singleton služeb** – konstruktor singleton volá databázi nebo soubor při startu celé aplikace. Přesuňte inicializaci do `IHostedService.StartAsync`.
- ❌ **Ignorování health check readiness** – aplikace přijme request dřív, než je zahřátá; první requesty jsou pomalé nebo selhávají.
- ❌ **Trimming bez testů** – trimmer odstraní typy, na které neexistuje statická reference (reflexe, JSON deserializace). Po zapnutí trimmingu spusťte celou test suite.
- ❌ **ReadyToRun bez `--self-contained`** – R2R obrazy jsou vázané na konkrétní verzi runtime; bez self-contained deploy nemusí platit na serveru s jinou verzí.

---

<details>
<summary>Deep dive: měření startup time a startup tracing</summary>

### Měření startup time

```bash
# Jednoduché měření přes time
time dotnet run --configuration Release

# Přesnější: EventPipe trace od spuštění procesu
dotnet-trace collect --process-id <pid> --providers Microsoft-Windows-DotNETRuntime:0x8:5
```

### Startup tracing v ASP.NET Core

```csharp
// Zapněte podrobné logování startu
builder.Logging.AddFilter("Microsoft.AspNetCore.Hosting", LogLevel.Debug);
builder.Logging.AddFilter("Microsoft.Extensions.DependencyInjection", LogLevel.Debug);
```

Log ukáže, kolik ms trvala každá fáze: DI registrace, middleware pipeline, hosted services.

### Vliv počtu NuGet balíčků

Každá přidaná assembly prodlouží startup o 1–50 ms (načtení, JIT cold start). Audit závislostí:

```bash
dotnet list package --outdated
# Zvažte: potřebujete 300 KB knihovnu pro jednu metodu?
```

Preferujte knihovny, které podporují NativeAOT a trimming – jsou navrženy pro minimální startup overhead.

</details>

**Další krok:** [JIT vs AOT kompilace](JIT_AOT.md)
