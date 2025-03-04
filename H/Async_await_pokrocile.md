> Tato stránka pokrývá **pokročilé vzory**. Pro úvod do async/await viz [B/Async_await.md](../B/Async_await.md).

# Pokročilé techniky s `async` a `await`

`async`/`await` je syntaktický cukr nad stavovým automatem, který .NET generuje při kompilaci. Pochopení toho, co se děje pod kapotou, vám pomůže psát kód, který je nejen správný, ale i výkonný.

## 1. Koncept

Když napíšete `await someTask`, kompilátor rozdělí metodu na dvě části: kód před `await` a kód po něm (pokračování). Pokud task ještě není dokončen, metoda vrátí řízení volajícímu a zaregistruje pokračování, které se spustí po dokončení. Vlákno tedy není blokováno – je uvolněno pro jinou práci.

Klíčové je pochopit, **co se stane po `await`**. Ve výchozím nastavení se pokračování spustí na původním synchronizačním kontextu (například na UI vlákně v WPF nebo WinForms). V serverovém kódu (ASP.NET Core) synchronizační kontext neexistuje, takže pokračování pokračuje na libovolném vlákně z ThreadPoolu. Právě tato asymetrie je zdrojem mnoha chyb.

## 2. Příklad

### Základní použití

```csharp
// HttpClient by měl být sdílený – viz IHttpClientFactory
public async Task<string> StahniDataAsync(string url, CancellationToken ct = default)
{
    using var client = new HttpClient();
    // GetStringAsync vrátí Task<string>; await uvolní vlákno, dokud nepřijde odpověď
    var data = await client.GetStringAsync(url, ct);
    return data;
}
```

### `async void` vs `async Task`

```csharp
// ❌ async void – výjimku nelze zachytit z volajícího kódu; nelze awaitovat; nelze testovat
private async void BtnKliknut_Click(object sender, EventArgs e)
{
    await NactiDataAsync(); // pokud vyhodí výjimku, aplikace padne bez možnosti zachycení
}

// ✅ async Task – výjimka putuje do Tasku, který volající může awaitovat
public async Task NactiDataAsync()
{
    await Task.Delay(500);
}

// ✅ async void je přijatelný POUZE pro event handlery – a i tam zabalte tělo do try/catch
private async void BtnKliknut_ClickBezpecne(object sender, EventArgs e)
{
    try { await NactiDataAsync(); }
    catch (Exception ex) { ZobrazChybu(ex); }
}
```

### Paralelní spuštění více úloh

```csharp
public async Task<string[]> StahniVsechnyAsync(string[] urls, CancellationToken ct = default)
{
    using var client = new HttpClient();

    // Spustíme všechny requesty najednou – žádné sekvenční čekání
    var tasky = urls.Select(url => client.GetStringAsync(url, ct));

    // WhenAll počká, až se dokončí všechny; pokud selže byť jeden, vyhodí AggregateException
    return await Task.WhenAll(tasky);
}
```

### Asynchronní inicializace objektu

Konstruktory nemohou být `async`. Standardní řešení je statická factory metoda:

```csharp
public class SluzbaSDatabazi
{
    private readonly DbConnection _conn;

    private SluzbaSDatabazi(DbConnection conn) => _conn = conn;

    public static async Task<SluzbaSDatabazi> VytvorAsync(string connectionString)
    {
        var conn = new SqlConnection(connectionString);
        await conn.OpenAsync();
        return new SluzbaSDatabazi(conn);
    }
}

var sluzba = await SluzbaSDatabazi.VytvorAsync(connectionString);
```

## 3. Kdy použít

- Všude, kde provádíte I/O operace (síť, soubory, databáze) – async uvolní vlákno po dobu čekání.
- Při paralelním spouštění nezávislých operací použijte `Task.WhenAll` místo sekvenčního `await`.
- Pokud spouštíte CPU-bound práci z UI vlákna, použijte `Task.Run`: `await Task.Run(() => TěžkýVýpočet())`.
- Vždy propagujte `CancellationToken` přes celý call stack – viz [CancellationToken.md](CancellationToken.md).

## 4. Časté chyby

- ❌ **`async void`** – výjimky z async void nejde zachytit z volajícího kódu; používejte `async Task`.
- ❌ **`task.Result` nebo `task.Wait()` v synchronním kódu** – na aplikacích se synchronizačním kontextem (WPF, WinForms, ASP.NET classic) způsobí deadlock. Volající zablokuje UI vlákno a čeká na pokračování, které se nikdy nespustí, protože vlákno je zablokované.
- ❌ **Fire-and-forget bez ošetření výjimek** – `_ = NactiDataAsync()` tiše pohltí výjimku. Pokud skutečně chcete fire-and-forget, logujte výjimky explicitně.
- ❌ **Vytváření nového `HttpClient` v každém requestu** – vyčerpá sockety. Používejte `IHttpClientFactory` nebo sdílenou instanci.
- ❌ **Ignorování `CancellationToken`** – metody, které nepřijímají `CancellationToken`, nelze zrušit; operace běží i po timeoutu nebo disconnectu klienta.
- ❌ **Sekvenční `await` ve smyčce nad nezávislými úlohami** – spusťte je paralelně přes `Task.WhenAll`.

---

<details>
<summary>Deep dive: stavový automat, ConfigureAwait a výkon</summary>

### Jak kompilátor transformuje async metodu

Každá `async` metoda je při kompilaci přeložena na třídu implementující `IAsyncStateMachine`. Metoda `MoveNext()` obsahuje celý kód metody rozsekaný na stavy oddělené `await` body. Při každém `await` se stav uloží a metoda se zaregistruje jako pokračování na awaitable objektu.

To má dopad na alokace: pro každou async metodu vzniká objekt na haldě (v mnoha případech .NET 5+ optimalizuje na stack). Právě proto existuje `ValueTask` – viz [Task_ValueTask.md](Task_ValueTask.md).

### ConfigureAwait(false)

```csharp
// V knihovním kódu (ne v UI vrstvě) vždy používejte ConfigureAwait(false)
var data = await client.GetStringAsync(url).ConfigureAwait(false);
```

`ConfigureAwait(false)` říká: po dokončení awaitu nechci pokračovat na původním synchronizačním kontextu – spusť mě na libovolném vlákně z ThreadPoolu. V ASP.NET Core to nemá efekt (synchronizační kontext neexistuje), ale v knihovnách je to dobrá praxe. Podrobněji viz [ConfigureAwait.md](ConfigureAwait.md).

### Sekvenční vs. paralelní await

```csharp
// Sekvenční – celkový čas = součet časů všech operací
var a = await ZiskejAAsync();
var b = await ZiskejBAsync();

// Paralelní – celkový čas = čas nejpomalejší operace
var taskA = ZiskejAAsync();
var taskB = ZiskejBAsync();
var (a, b) = (await taskA, await taskB);
// nebo:
var vysledky = await Task.WhenAll(taskA, taskB);
```

### Výjimky v Task.WhenAll

```csharp
var vsechnyUlohy = Task.WhenAll(task1, task2, task3);
try
{
    await vsechnyUlohy;
}
catch
{
    // Catch zachytí pouze první výjimku.
    // Pro všechny výjimky přistupujte přes .Exception.InnerExceptions:
    foreach (var ex in vsechnyUlohy.Exception!.InnerExceptions)
        Console.WriteLine(ex.Message);
}
```

</details>

---

**Související kapitoly:**
- [Task a ValueTask](Task_ValueTask.md) – kdy použít `ValueTask` místo `Task`
- [ConfigureAwait](ConfigureAwait.md) – synchronizační kontext podrobně
- [CancellationToken](CancellationToken.md) – správné zrušení operací
- [Task orchestrace](Task_Orchestration.md) – `WhenAll`, `WhenAny`, timeouty
- [IAsyncEnumerable](IAsyncEnumerable.md) – asynchronní streamy dat
- [TaskCompletionSource](TaskCompletionSource.md) – manuální řízení Tasku

**Další krok:** [Správné použití Task a ValueTask](Task_ValueTask.md)
