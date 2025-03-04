# Cache a strategie ukládání do mezipaměti

Cache je vyrovnávací vrstva, která ukládá výsledky drahých operací (databázové dotazy, výpočty, HTTP volání) tak, aby je bylo možné opakovaně použít bez opakovaného volání zdroje. Správná strategie cache dokáže snížit latenci requestů z stovek milisekund na zlomky milisekund a dramaticky odlehčit databázi.

## 1. Koncept

Základní otázka při návrhu cache není "jak cacheovat", ale "co cacheovat a jak dlouho". Data s krátkou životností (aktuální ceny, dostupnost skladem) potřebují krátkou expiraci nebo aktivní invalidaci. Data, která se mění zřídka (katalog produktů, konfigurace), mohou být cachována desítky minut nebo i hodiny.

### IMemoryCache vs IDistributedCache

| Vlastnost | `IMemoryCache` | `IDistributedCache` (Redis) |
|-----------|---------------|----------------------------|
| Umístění dat | RAM aktuálního procesu | Sdílená (Redis, SQL Server, ...) |
| Rychlost | ~mikrosekundy | ~sub-milisekunda (síť) |
| Sdílení mezi instancemi | ❌ Každá instance má svou cache | ✅ Všechny instance sdílejí |
| Přežití restartu | ❌ | ✅ (Redis s persistencí) |
| Složitost nasazení | Nulová | Vyžaduje Redis/SQL |

Pro jednoduché aplikace na jednom serveru stačí `IMemoryCache`. Pro Kubernetes, load balancing nebo serverless potřebujete `IDistributedCache`.

## 2. Příklad

### IMemoryCache – základní použití

```csharp
public class ProduktSluzba(IMemoryCache cache, IProduktRepo repo)
{
    public async Task<Produkt?> ZiskejProduktAsync(int id, CancellationToken ct = default)
    {
        // GetOrCreateAsync: atomicky zkontroluje cache a pokud chybí, zavolá factory
        return await cache.GetOrCreateAsync($"produkt_{id}", async entry =>
        {
            // Absolute expiration = 10 minut od vložení, bez ohledu na přístupy
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10);
            // SlidingExpiration = posune expiraci o 2 minuty při každém přístupu
            entry.SlidingExpiration = TimeSpan.FromMinutes(2);
            // Priorita určuje, co se odstraní první při tlaku na paměť
            entry.Priority = CacheItemPriority.Normal;
            return await repo.NactiAsync(id, ct);
        });
    }

    // Explicitní invalidace po změně
    public async Task AktualizujProduktAsync(Produkt produkt, CancellationToken ct = default)
    {
        await repo.UlozAsync(produkt, ct);
        cache.Remove($"produkt_{produkt.Id}"); // Odstraní zastaralou položku
    }
}
```

### IDistributedCache s Redis

```csharp
// Program.cs
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
    options.InstanceName = "MojeApp:";
});

// Použití
public class SeznamKategoriiSluzba(IDistributedCache cache, IKategorieRepo repo)
{
    private static readonly JsonSerializerOptions _json = new(JsonSerializerDefaults.Web);

    public async Task<List<Kategorie>> ZiskejAsync(CancellationToken ct = default)
    {
        const string klic = "kategorie_vsechny";

        var cachovana = await cache.GetStringAsync(klic, ct);
        if (cachovana is not null)
            return JsonSerializer.Deserialize<List<Kategorie>>(cachovana, _json)!;

        var data = await repo.NactiVsechnyAsync(ct);

        await cache.SetStringAsync(klic,
            JsonSerializer.Serialize(data, _json),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1)
            }, ct);

        return data;
    }
}
```

### Cache-Aside (nejběžnější vzor)

```csharp
// Vzor: 1) zkontroluj cache, 2) pokud chybí, načti zdroj, 3) ulož do cache
public async Task<Report> ZiskejReportAsync(DateOnly datum, CancellationToken ct)
{
    var klic = $"report_{datum:yyyyMMdd}";

    if (cache.TryGetValue(klic, out Report? cachovany))
        return cachovany!;

    var report = await reportGenerator.GenerujAsync(datum, ct); // drahá operace

    cache.Set(klic, report, new MemoryCacheEntryOptions
    {
        AbsoluteExpiration = datum.ToDateTime(TimeOnly.MinValue).AddDays(1)
        // Report pro minulý den expiruje o půlnoci – nepotřebuje invalidaci
    });

    return report;
}
```

## 3. Kdy použít

- **`IMemoryCache`** – jednoduchá aplikace na jednom serveru; data, která stačí mít na úrovni procesu (konfigurace, lookup tabulky, výsledky výpočtů).
- **`IDistributedCache` (Redis)** – více instancí aplikace (load balancer, Kubernetes); session data; data, která musí přežít restart procesu.
- **Absolute expiration** – data s pevnou dobou platnosti (zpravodajské články, ceníky platné do konce dne).
- **Sliding expiration** – data, která by měla zůstat v cache, dokud jsou aktivně používána (profil přihlášeného uživatele).
- **Žádná cache** – data, která se mění po každém requestu, nebo kde nekonzistence způsobuje chyby (finanční transakce, inventář v reálném čase).

## 4. Časté chyby

- ❌ **Cache stampede** – desítky vláken najednou zjistí, že cache je prázdná, a všechna zavolají databázi. Řešte `SemaphoreSlim` nebo lazy initialization.
- ❌ **Cacheování null nebo chybových výsledků** – `GetOrCreateAsync` uloží `null` jako platnou hodnotu; pokud databáze přechodně selže, nacachujete chybu. Explicitně kontrolujte výsledek.
- ❌ **Příliš dlouhá expirace pro často se měnící data** – po dobu expirace uvidí uživatelé zastaralá data; nastavte expiraci podle skutečné frekvence změn.
- ❌ **Klíče bez prefixu instance** – v distribuované cache více aplikací na stejném Redis sdílí klíče; vždy používejte prefix (`InstanceName` v Redis konfiguraci).
- ❌ **`IMemoryCache` v Kubernetes** – každý pod má vlastní paměť; po restartu nebo scale-out je cache prázdná; přejděte na Redis.

---

<details>
<summary>Deep dive: cache strategie, eviction a Redis výkonové charakteristiky</summary>

### Strategie expirace

**Absolute expiration** je jednoduchá a předvídatelná, ale může způsobit "cliff effect": v okamžiku expirace všechny přístupy padnou na databázi najednou. Pokud máte milion uživatelů a cache expiruje každých 5 minut, každých 5 minut dostanete špičkový nápor.

**Sliding expiration** tento problém řeší pro aktivně používaná data – dokud je někdo čte, zůstávají v cache. Riziko: populární data se nikdy nevymaní a mohou být zastaralá velmi dlouho. Kombinujte s absolutní expirací jako pojistkou.

### Eviction policies v MemoryCache

Při tlaku na paměť .NET MemoryCache odstraňuje položky dle priority:

```csharp
entry.Priority = CacheItemPriority.Low;       // Odstraní první
entry.Priority = CacheItemPriority.Normal;    // Výchozí
entry.Priority = CacheItemPriority.High;      // Odstraní poslední
entry.Priority = CacheItemPriority.NeverRemove; // Neodstraní (pozor na OOM)
```

### Redis – výkon a latence

Redis je in-memory databáze, která zvládá 100 000+ operací za sekundu na jednom nodu. Typická latence přes localhost je < 0.5 ms; přes síť ve stejném datacentru 1–2 ms. Pro srovnání: PostgreSQL query s indexem trvá 1–10 ms, bez indexu řádově víc.

Redis pipeline (dávkové odesílání příkazů) snižuje round-trip overhead:

```csharp
// StackExchange.Redis – pipeline
var db = redis.GetDatabase();
var batch = db.CreateBatch();
var tasks = kliče.Select(k => batch.StringGetAsync(k)).ToList();
batch.Execute();
var vysledky = await Task.WhenAll(tasks);
```

### Write-Through a Write-Behind

**Write-through**: při každém zápisu do databáze aktualizujte i cache atomicky. Garantuje konzistenci, ale přidává latenci zápisů.

**Write-behind (write-back)**: zapište nejdříve do cache, asynchronně přeneste do databáze. Rychlejší zápisy, ale riskujete ztrátu dat při výpadku.

Pro většinu webových aplikací je **cache-aside** (lazy loading) s krátkou expirací nejjednodušší a dostatečně spolehlivý kompromis.

</details>

**Další krok:** [Správa paměti a Garbage Collection](Sprava_pameti.md)
