# Práce s databázemi v cloudu

## SQL Server / Azure SQL s Entity Framework Core

Entity Framework Core (EF Core) je ORM – mapuje C# třídy na databázové tabulky.

```bash
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Tools
```

```csharp
// Model
public class Produkt
{
    public int Id { get; set; }
    public string Nazev { get; set; }
    public decimal Cena { get; set; }
}

// DbContext
public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }
    public DbSet<Produkt> Produkty { get; set; }
}
```

```csharp
// Registrace v Program.cs
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Default")));
```

```csharp
// CRUD operace
await _db.Produkty.AddAsync(new Produkt { Nazev = "Chléb", Cena = 29.90m });
await _db.SaveChangesAsync();

var levne = await _db.Produkty
    .Where(p => p.Cena < 50)
    .OrderBy(p => p.Nazev)
    .ToListAsync();
```

## Migrace schématu

```bash
dotnet ef migrations add PrvniMigrace
dotnet ef database update
```

## Azure Cosmos DB (NoSQL)

```bash
dotnet add package Microsoft.Azure.Cosmos
```

```csharp
CosmosClient client = new CosmosClient(connectionString);
Container container = client.GetContainer("MojeDB", "Produkty");

// Zapis
var produkt = new { id = "1", Nazev = "Chléb", Cena = 29.90 };
await container.CreateItemAsync(produkt, new PartitionKey("1"));

// Čtení
var response = await container.ReadItemAsync<dynamic>("1", new PartitionKey("1"));
```

## Azure Cache for Redis

```csharp
// NuGet: StackExchange.Redis
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration["Redis:ConnectionString"];
});

// Použití
IDistributedCache cache = ...;
await cache.SetStringAsync("klic", "hodnota", new DistributedCacheEntryOptions
{
    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10)
});
string? hodnota = await cache.GetStringAsync("klic");
```

## Výběr produkční databáze

| Databáze | Typ | Kdy použít |
|---|---|---|
| Azure SQL | Relační (SQL) | Transakce, složité dotazy |
| Cosmos DB | NoSQL (dokument) | Globální distribuce, flexibilní schéma |
| Azure Cache (Redis) | Klíč-hodnota | Cache, sessions, pub/sub |
| Azure Table Storage | NoSQL (tabulka) | Levné uložení strukturovaných dat |

---

**Další krok:** [Azure Storage – Blob, Queue, Table](Azure_Storage.md)
