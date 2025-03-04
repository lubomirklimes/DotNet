# Orchestrace úloh: Task.WhenAll, Task.WhenAny a timeouty

Když potřebujete spustit více asynchronních operací najednou, nestačí sekvenční `await`. .NET nabízí nástroje pro koordinaci paralelních tasků: čekat na všechny, reagovat na první, nebo zavést timeout.

## 1. Koncept

### Task.WhenAll – všechny musí dokončit

`Task.WhenAll` přijme kolekci tasků a vrátí task, který se dokončí teprve poté, co se dokončí **všechny** vstupní tasky. Tasky přitom **běží paralelně** – nespouštíte je sekvenčně.

### Task.WhenAny – stačí jeden

`Task.WhenAny` se dokončí, jakmile se dokončí **první** z předaných tasků. Ostatní tasky **stále běží** – WhenAny je nezruší. Typické použití: timeout (závodění operace s `Task.Delay`), nebo "vezmi první dostupný výsledek".

### Proč ne sekvenční await

```csharp
// Celkový čas = součet časů = 3s
var a = await ZiskejAsync("A"); // čeká 1s
var b = await ZiskejAsync("B"); // čeká 1s
var c = await ZiskejAsync("C"); // čeká 1s

// WhenAll: celkový čas = nejpomalejší = 1s
// Varianta 1: WhenAll vrací pole T[] – indexování
var vysledky = await Task.WhenAll(ZiskejAsync("A"), ZiskejAsync("B"), ZiskejAsync("C"));
var (a, b, c) = (vysledky[0], vysledky[1], vysledky[2]);

// Varianta 2: spusť tasky zvlášť, pak awaítuj najednou (čitelnější)
var taskA = ZiskejAsync("A"); var taskB = ZiskejAsync("B"); var taskC = ZiskejAsync("C");
await Task.WhenAll(taskA, taskB, taskC);
var (a2, b2, c2) = (await taskA, await taskB, await taskC);
```

## 2. Příklad

### WhenAll – paralelní načítání dat

```csharp
public async Task<DashboardData> NactiDashboardAsync(int userId, CancellationToken ct)
{
    // Všechny tři requesty spustíme najednou
    var profilTask    = _userRepo.NactiProfilAsync(userId, ct);
    var objednavkyTask = _objednavkyRepo.NactiAsync(userId, ct);
    var notifikaceTask = _notifikaceRepo.NactiAsync(userId, ct);

    // Počkáme na všechny – nejdéle trvá ten nejpomalejší
    await Task.WhenAll(profilTask, objednavkyTask, notifikaceTask);

    return new DashboardData(
        Profil: profilTask.Result,        // .Result je bezpečné PO awaitu WhenAll
        Objednavky: objednavkyTask.Result,
        Notifikace: notifikaceTask.Result
    );
}
```

### WhenAll se zpracováním výsledků

```csharp
public async Task<string[]> StahniVsechnyAsync(string[] urls, CancellationToken ct)
{
    using var client = new HttpClient();
    // Select vrátí IEnumerable<Task<string>> – tasky jsou spuštěny okamžitě
    var tasky = urls.Select(url => client.GetStringAsync(url, ct));
    return await Task.WhenAll(tasky); // string[] ve stejném pořadí jako urls
}
```

### Ošetření chyb při WhenAll

```csharp
var vsechnyUlohy = Task.WhenAll(task1, task2, task3);
try
{
    await vsechnyUlohy;
}
catch
{
    // catch zachytí pouze PRVNÍ výjimku z AggregateException
    // Pro všechny výjimky:
    if (vsechnyUlohy.IsFaulted)
    {
        foreach (var ex in vsechnyUlohy.Exception!.InnerExceptions)
            log.LogError(ex, "Úloha selhala");
    }
}
```

### WhenAny – timeout pattern

```csharp
public async Task<string?> StahniSTimeoutemAsync(string url, TimeSpan timeout, CancellationToken ct)
{
    using var client = new HttpClient();
    var stahovaniTask = client.GetStringAsync(url, ct);
    var timeoutTask   = Task.Delay(timeout, ct);

    var prvniDokonceny = await Task.WhenAny(stahovaniTask, timeoutTask);

    if (prvniDokonceny == timeoutTask)
    {
        // Timeout nastal dřív – stahování stále běží na pozadí
        // Pro čistý cancel použijte raději CancellationTokenSource.CancelAfter
        return null;
    }

    return await stahovaniTask; // Bezpečně získáme výsledek (nebo výjimku)
}

// ✅ Elegantnější přístup přes CancellationToken
public async Task<string> StahniSTimeoutemCTAsync(string url, CancellationToken ct)
{
    using var timeoutCts = CancellationTokenSource.CreateLinkedTokenSource(ct);
    timeoutCts.CancelAfter(TimeSpan.FromSeconds(10));

    using var client = new HttpClient();
    return await client.GetStringAsync(url, timeoutCts.Token);
    // OperationCanceledException pokud timeout nebo ct zruší
}
```

### WhenAny – první dostupný výsledek (hedging)

```csharp
// Hedging: pošli request na dvě repliky, vezmi rychlejší odpověď
public async Task<string> StahniZNejrychlejsihoAsync(string[] repliky, CancellationToken ct)
{
    var tasky = repliky.Select(url =>
    {
        using var client = new HttpClient();
        return client.GetStringAsync(url, ct);
    }).ToList();

    while (tasky.Count > 0)
    {
        var dokonceny = await Task.WhenAny(tasky);
        if (!dokonceny.IsFaulted)
            return await dokonceny; // Úspěšný výsledek
        tasky.Remove(dokonceny);    // Selhání – zkus další
    }

    throw new Exception("Všechny repliky selhaly.");
}
```

## 3. Kdy použít

- **`Task.WhenAll`** – načítání nezávislých dat pro sestavení odpovědi (dashboard, report); paralelní zpracování kolekce dat.
- **`Task.WhenAny` + timeout** – pro jednoduchou timeout logiku (ale preferujte `CancellationToken` s `CancelAfter`).
- **`Task.WhenAny` + hedging** – kritické operace s více replikami; vezměte první úspěšnou odpověď.

## 4. Časté chyby

- ❌ **Přístup k `task.Result` před `WhenAll`** – task nemusí být dokončen; vždy awaítujte WhenAll nejdříve.
- ❌ **Záměna WhenAll a WhenAny** – WhenAll čeká na všechny; WhenAny na první. Záměna způsobí předčasné pokračování nebo zbytečné čekání.
- ❌ **WhenAny bez zrušení ostatních tasků** – WhenAny nezruší zbývající tasky; pokud je necháte běžet, mohou spotřebovávat zdroje. Pokud je nepotřebujete, předejte `CancellationToken` a po WhenAny ho zrušte.
- ❌ **Ignorování výjimek v WhenAll** – pokud `await Task.WhenAll(...)` vyhodí výjimku a nezachytíte ji, ostatní výjimky jsou tiše zahozeny.

---

<details>
<summary>Deep dive: WaitAll vs WhenAll a výkonové aspekty</summary>

### Task.WaitAll vs Task.WhenAll

```csharp
Task.WaitAll(task1, task2); // Synchronní blokování – blokuje vlákno!
await Task.WhenAll(task1, task2); // Asynchronní – uvolní vlákno
```

`Task.WaitAll` blokuje vlákno ThreadPoolu po celou dobu čekání. V ASP.NET Core toto sníží propustnost. V UI aplikaci způsobí freeznutí. Vždy preferujte `await Task.WhenAll`.

### Výkon při mnoha taskcích

Při stovkách tasků v `WhenAll` pozor na alokace. Každý `async` lambda v `Select` alokuje state machine. Pro velké kolekce zvažte `Parallel.ForEachAsync`:

```csharp
// Pro velké kolekce s omezením paralelismu
await Parallel.ForEachAsync(polozky,
    new ParallelOptions { MaxDegreeOfParallelism = 10, CancellationToken = ct },
    async (polozka, token) => await ZpracujAsync(polozka, token));
```

</details>

**Další krok:** [IAsyncEnumerable a asynchronní streamování dat](IAsyncEnumerable.md)
