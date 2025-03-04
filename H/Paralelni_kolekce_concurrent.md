# Paralelní kolekce v .NET – ConcurrentQueue, ConcurrentDictionary a další

Standardní kolekce (`List<T>`, `Dictionary<TKey,TValue>`, `Queue<T>`) nejsou thread-safe: souběžné čtení a zápis bez synchronizace vede k poškození dat nebo výjimkám. `System.Collections.Concurrent` nabízí kolekce navržené přímo pro vícevláknové prostředí – bez hrubého `lock` na celou kolekci.

## 1. Koncept

Klíčový rozdíl oproti ruční synchronizaci (`lock + List<T>`) je granularita zámků. `ConcurrentDictionary<TKey,TValue>` například uzamyká pouze příslušný segment (bucket), ne celý slovník – více vláken může pracovat s různými klíči skutečně souběžně. `ConcurrentQueue<T>` a `ConcurrentStack<T>` používají lock-free algoritmy postavené na CAS (Compare-And-Swap) CPU instrukcích.

Volba správné kolekce závisí na třech otázkách:
1. Záleží na pořadí zpracování (FIFO, LIFO, nebo nezáleží)?
2. Potřebuji atomické operace read-modify-write (jako `AddOrUpdate`)?
3. Potřebuji neměnný (immutable) snapshot pro bezpečné čtení?

## 2. Příklad

### ConcurrentQueue – neblokující FIFO fronta

```csharp
var fronta = new ConcurrentQueue<LogZaznam>();

// Více vláken může přidávat souběžně
Parallel.For(0, 1000, i => fronta.Enqueue(new LogZaznam(i)));

// Konzumace – TryDequeue vrátí false, pokud je fronta prázdná (neblokuje)
while (fronta.TryDequeue(out var zaznam))
    UlozZaznam(zaznam);

Console.WriteLine($"Zbývá: {fronta.Count}"); // 0
```

### ConcurrentDictionary – atomické operace

```csharp
var citace = new ConcurrentDictionary<string, int>();

// GetOrAdd je atomické – bez race condition při prvním přidání
var hodnota = citace.GetOrAdd("klic", 0);

// AddOrUpdate – přidej nebo aktualizuj atomicky
// Pozor: factory funkce může být volána vícekrát; nesmí mít side-effecty
citace.AddOrUpdate(
    key: "navstev",
    addValue: 1,
    updateValueFactory: (key, stara) => stara + 1
);

// Typický vzor pro čítač přístupů
void ZaznamenejPristup(string stranku)
    => _citace.AddOrUpdate(stranku, 1, (_, stara) => stara + 1);
```

### ConcurrentStack – LIFO zásobník

```csharp
var zasobnik = new ConcurrentStack<int>();

// PushRange je efektivnější než opakované Push
zasobnik.PushRange(new[] { 1, 2, 3, 4, 5 });

if (zasobnik.TryPop(out int vrchol))
    Console.WriteLine($"Vrchol: {vrchol}"); // 5

// PopRange – odebrání více položek najednou
var batch = new int[3];
int odebrano = zasobnik.TryPopRange(batch);
```

### ConcurrentBag – neuspořádaná kolekce

```csharp
// ConcurrentBag je optimalizován pro scénář, kde vlákno přidává I odebírá ze stejné kolekce
// Každé vlákno má vlastní lokální frontu; přístup k cizí frontě je méně časté
var bag = new ConcurrentBag<WorkItem>();

Parallel.ForEach(zdroj, polozka =>
{
    var vysledek = Zpracuj(polozka);
    bag.Add(vysledek); // vlákno přidává do své lokální fronty
});

// Pořadí výsledků není zaručeno
foreach (var item in bag)
    Console.WriteLine(item);
```

### Immutable Collections – thread-safe čtení bez zámků

```csharp
using System.Collections.Immutable;

// ImmutableList je thread-safe pro čtení z libovolného počtu vláken
var seznam = ImmutableList.Create("a", "b", "c");

// Modifikace vrátí NOVOU instanci; původní seznam se nezmění
var rozsirenySeznam = seznam.Add("d");

// Pattern pro sdílenou konfiguraci: atomická záměna reference
// Interlocked.CompareExchange zajistí paměťovou bariéru – volatile je nadbytečné
private ImmutableDictionary<string, string> _konfig
    = ImmutableDictionary<string, string>.Empty;

public void AktualizujKonfiguraci(string klic, string hodnota)
{
    // Interlocked.Exchange zajistí atomické přepsání reference
    ImmutableDictionary<string, string> stara, nova;
    do {
        stara = _konfig;
        nova = stara.SetItem(klic, hodnota);
    } while (Interlocked.CompareExchange(ref _konfig, nova, stara) != stara);
}
```

## 3. Kdy použít

| Kolekce | Pořadí | Ideální scénář |
|---------|--------|----------------|
| `ConcurrentQueue<T>` | FIFO | Fronty úloh, logování, event buffery |
| `ConcurrentStack<T>` | LIFO | Undo zásobníky, rekurzivní zpracování |
| `ConcurrentBag<T>` | Žádné | Sběr výsledků z paralelní smyčky |
| `ConcurrentDictionary<K,V>` | N/A | Cache, čítače přístupů, sdílený stav |
| `ImmutableList/Dictionary` | N/A | Sdílená konfigurace, snapshot dat |

## 4. Časté chyby

- ❌ **`ConcurrentDictionary` jako náhrada `lock + Dictionary` pro složitý read-modify-write** – `AddOrUpdate` factory funkce může být volána vícekrát (pokud dvě vlákna soutěží o stejný klíč). Factory nesmí mít side-effecty (I/O, volání databáze).
- ❌ **`ConcurrentBag` pro FIFO zpracování** – pořadí není garantováno; pro FIFO použijte `ConcurrentQueue`.
- ❌ **Čtení `Count` pro kontrolu, zda odebrat položku** – mezi `Count > 0` a `TryDequeue` může jiné vlákno položku odebrat. Vždy používejte `TryDequeue`/`TryPop` přímo.
- ❌ **Immutable kolekce pro časté zápisy** – každá změna alokuje novou instanci; při tisících zápisů za sekundu to přetíží GC. Pro časté zápisy použijte `ConcurrentDictionary`.

---

<details>
<summary>Deep dive: lock-free algoritmy a výkonové charakteristiky</summary>

### Jak funguje lock-free ConcurrentQueue

`ConcurrentQueue<T>` v .NET 5+ je implementována jako segmentovaný ring-buffer. Každý segment je pole s head a tail indexy. Přidání položky je CAS operace na tail; odebrání je CAS na head. Pokud dvě vlákna soutěží o tail, jedno uspěje a druhé zopakuje pokus – žádný lock, žádné přepnutí kontextu.

Oproti `lock + Queue<T>` je ConcurrentQueue výrazně rychlejší při nízké kontenci (málo soutěžících vláken). Při velmi vysoké kontenci (desítky vláken neustále přidávají/odebírají) se výkon přibližuje, ale nikdy neklesne pod lock-based implementaci.

### ConcurrentDictionary segmentace

Interně je rozdělena na segmenty (buckets). Zámek se drží pouze na příslušný segment, ne na celý slovník. Výchozí počet segmentů je `Environment.ProcessorCount * 4`. Přizpůsobit lze přes konstruktor:

```csharp
// concurrencyLevel = počet vláken, která budou souběžně zapisovat
var dict = new ConcurrentDictionary<string, int>(
    concurrencyLevel: Environment.ProcessorCount,
    capacity: 1000);
```

### Immutable Collections a structural sharing

`ImmutableList<T>` je implementována jako AVL strom. Přidání prvku nevytváří novou kopii celého seznamu, ale sdílí většinu uzlů stromu s původní verzí (structural sharing). Paměťová stopa přírůstku je O(log n), ne O(n). Pro velké seznamy je to přijatelné; pro malé seznamy (< 100 prvků) je `ImmutableArray<T>` (prostý immutable array) efektivnější.

</details>