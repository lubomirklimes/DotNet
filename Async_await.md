### Pokročilé techniky s `async` a `await` v C# a .NET: Co potřebují seniorní vývojáři vědět

Asynchronní programování se v .NET a C# stalo nepostradatelným nástrojem pro vývoj škálovatelných a efektivních aplikací. Klíčovými stavebními bloky této techniky jsou klíčová slova `async` a `await`, 
která usnadňují práci s asynchronními úlohami a umožňují provádět neblokující operace. Pro zkušené vývojáře, kteří chtějí maximalizovat výkon svých aplikací a vyvarovat se běžných chyb, 
však existuje řada pokročilých technik a konceptů, které jdou nad rámec základního použití těchto klíčových slov.

#### 1\. **Základy `async` a `await` - připomenutí pro pokročilé**

Asynchronní programování umožňuje provádění dlouhotrvajících operací, aniž by se blokovala hlavní vlákna. Když použijete klíčová slova `async` a `await`, operace se rozdělí na dvě fáze:

-   **První část**: Úloha se spustí a okamžitě vrátí řízení volající metodě, zatímco operace probíhá asynchronně.
-   **Druhá část**: Po dokončení úlohy se asynchronní metoda obnoví a pokračuje ve svém běhu od místa, kde byla přerušena, aniž by blokovala hlavní vlákno.

Příklad základního použití:

```
public async Task DownloadFileAsync(string url)
{
    using var client = new HttpClient();
    var data = await client.GetStringAsync(url);
    Console.WriteLine(data);
}
```

Tato syntaxe je snadno čitelná, ale pokročilí vývojáři často potřebují řešit problémy, které základní přístup nezahrnuje, jako je optimalizace výkonu, zpracování chyb nebo paralelizace úloh.

#### 2\. **Pokročilé techniky s `async` a `await`**

##### a) **Zamezení `async void`: Používejte `Task`**

Jedním z nejčastějších problémů v asynchronním programování je používání návratového typu `async void`. Tento návratový typ by měl být používán **pouze** v případě událostí, protože nepodporuje správné zpracování výjimek a nemá možnost zpětné vazby o dokončení operace. Místo toho byste měli vždy preferovat návratový typ `Task` (nebo `Task<T>`pro metody vracející hodnotu).

-   **Správně:**

    ```
    public async Task DoWorkAsync()
    {
        await Task.Delay(1000);
    }
    ```

-   **Špatně:**

    ```
    public async void DoWorkAsync() // Nebezpečné!
    {
        await Task.Delay(1000);
    }
    ```

##### b) **Použití `ConfigureAwait(false)`**

Klíčovým aspektem asynchronního programování je **obnovení kontextu po await**. Když voláte asynchronní metodu a použijete `await`, běh aplikace se po dokončení asynchronní operace obnoví v původním kontextu, což může být např. uživatelské rozhraní (UI thread) nebo synchronizační kontext. Obnovení kontextu může být v některých scénářích zbytečné a může vést k výraznému zpomalení, zejména u serverových aplikací.

Řešení je volat `ConfigureAwait(false)` u všech operací, kde nepotřebujete vrátit kontrolu zpět na původní kontext.

```
public async Task DownloadFileAsync(string url)
{
    using var client = new HttpClient();
    var data = await client.GetStringAsync(url).ConfigureAwait(false); // Zabrání obnovení kontextu
    // Pokračujte bez návratu na původní vlákno
}
```

##### c) **Zpracování výjimek v asynchronním kódu**

Výjimky v asynchronním kódu se chovají trochu jinak než ve standardním synchronním kódu. Pokud dojde k výjimce v metodě označené `async`, tato výjimka je zabalena do objektu `Task` a musí být správně zpracována pomocí `await`.

**Příklad:**

```
public async Task DownloadFileAsync(string url)
{
    try
    {
        using var client = new HttpClient();
        var data = await client.GetStringAsync(url);
        Console.WriteLine(data);
    }
    catch (HttpRequestException ex)
    {
        Console.WriteLine($"Request failed: {ex.Message}");
    }
}
```

Výjimky, které nejsou zpracovány pomocí `await`, budou ignorovány, což může vést k neodhaleným problémům. Proto je důležité zajistit, aby všechny asynchronní operace byly správně `awaitované`.

##### d) **Asynchronní LINQ s `Select` a `Where`**

Asynchronní operace mohou být kombinovány s LINQ pro výkonnou práci s datovými kolekcemi. Místo tradičního synchronního LINQ můžete používat asynchronní verzi pomocí metody `Select`, která využívá `await`.

**Příklad asynchronního LINQ:**

```
var urls = new[] { "https://example.com", "https://google.com" };
var tasks = urls.Select(async url =>
{
    using var client = new HttpClient();
    return await client.GetStringAsync(url);
});

var results = await Task.WhenAll(tasks);
```

Zde každá operace `GetStringAsync` probíhá paralelně, aniž by čekala na dokončení předchozí, což zajišťuje efektivní využití zdrojů.

##### e) **Paralelizace úloh s `Task.WhenAll` a `Task.WhenAny`**

Často budete chtít spustit několik asynchronních operací paralelně a počkat, až se všechny dokončí. Pro tyto scénáře jsou klíčové metody `Task.WhenAll` a `Task.WhenAny`.

-   **`Task.WhenAll`:** Spuštění více úloh paralelně a čekání, dokud nejsou všechny dokončeny.

    ```
    public async Task DownloadFilesAsync(string[] urls)
    {
        var tasks = urls.Select(url => DownloadFileAsync(url));
        await Task.WhenAll(tasks); // Čeká, dokud nejsou všechny úlohy dokončeny
    }
    ```

-   **`Task.WhenAny`:** Čeká na dokončení jakékoli z úloh, což je užitečné v situacích, kdy chcete rychlou odpověď a nezáleží na tom, která operace se dokončí jako první.

    ```
    public async Task<string> GetFirstCompletedFileAsync(string[] urls)
    {
        var tasks = urls.Select(url => DownloadFileAsync(url));
        var completedTask = await Task.WhenAny(tasks);
        return await completedTask; // Vrátí výsledek první dokončené úlohy
    }
    ```

##### f) **Zajištění, že asynchronní úlohy neskončí v deadlocku**

Jedním z častých problémů v asynchronním programování je deadlock, ke kterému může dojít, pokud použijete `Task.Result` nebo `Task.Wait()` ve vláknech, která používají synchronizační kontext. To může způsobit, že se úloha nikdy nedokončí, protože čekající kód zablokuje vlákno, na kterém by se úloha měla dokončit.

-   **Příklad deadlocku:**

    ```
    public void Run()
    {
        var task = DownloadFileAsync("https://example.com");
        task.Wait(); // Může způsobit deadlock
    }
    ```

Řešení je **vždy používat `await`** místo synchronního čekání na výsledek:

```
public async Task RunAsync()
{
    await DownloadFileAsync("https://example.com"); // Bezpečné
}
```

##### g) **Správa cancelací s `CancellationToken`**

Asynchronní metody by měly podporovat zrušení (cancellation) pomocí `CancellationToken`. Tento token umožňuje volajícímu procesu zrušit operaci, pokud už není potřeba, což šetří prostředky.

-   **Použití `CancellationToken`:**

    ```
    public async Task DownloadFileAsync(string url, CancellationToken cancellationToken)
    {
        using var client = new HttpClient();
        var data = await client.GetStringAsync(url, cancellationToken);
        Console.WriteLine(data);
    }
    ```

-   **Spuštění s `CancellationTokenSource`:**

    ```
    var cts = new CancellationTokenSource();
    var task = DownloadFileAsync("https://example.com", cts.Token);

    // Zrušení operace
    cts.Cancel();
    ```

##### h) **Asynchronní inicializace objektů**

V určitých případech může být nutné inicializovat objekty asynchronně. To však nelze provést přímo v konstruktoru, protože konstruktor v C# nemůže být asynchronní. Řešením je použití tzv. "asynchronních továren" nebo asynchronních inicializačních metod.

**Příklad asynchronní inicializace:**

```
public class MyAsyncClass
{
    private MyAsyncClass() { }

    public static async Task<MyAsyncClass> CreateAsync()
    {
        var instance = new MyAsyncClass();
        await instance.InitializeAsync();
        return instance;
    }

    private async Task InitializeAsync()
    {
        // Asynchronní inicializační logika
        await Task.Delay(1000);
    }
}
```

#### 3\. **Závěr**

Pokročilé asynchronní programování v .NET a C# vyžaduje nejen znalost základních konceptů jako `async` a `await`, ale také hlubší pochopení mechanismů jako `ConfigureAwait`, zpracování výjimek, paralelizace úloh, správné správy `CancellationToken` a vyhnutí se deadlockům. Pro seniorní vývojáře je zvládnutí těchto technik klíčové pro vytváření vysoce výkonných, škálovatelných a efektivních aplikací.

Využití těchto pokročilých technik vám pomůže psát kód, který je optimalizovaný pro asynchronní operace, minimalizuje riziko chyb a maximalizuje výkon aplikace v různých scénářích, ať už se jedná o desktopové aplikace nebo serverové systémy na vysoké úrovni zatížení.
