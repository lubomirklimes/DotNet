# Pokročilé použití `TaskCompletionSource`

`TaskCompletionSource<T>` umožňuje **explicitně řídit dokončení `Task`** – vhodné tam, kde nemůžeme použít `async/await` přímo: adaptace callback API, jednorázové události, bridging mezi synchronním a asynchronním světem.

## 1. Koncept

`TaskCompletionSource<T>` (TCS) je factory pro `Task<T>`, jehož výsledek nastavíte ručně voláním `SetResult`, `SetException`, nebo `SetCanceled`. Task pak lze předat volajícímu jako běžný awaitable.

Klíčový parametr: `TaskCreationOptions.RunContinuationsAsynchronously` zajistí, že pokračování awaiterů nespouští přímo na vlákně, které zavolá `SetResult` – zabrání deadlockům a neočekávaným výkonovým propadům.

```csharp
// Vždy preferujte RunContinuationsAsynchronously
var tcs = new TaskCompletionSource<string>(TaskCreationOptions.RunContinuationsAsynchronously);
```

## 2. Příklad

### Adaptace callback API na Task

Starší knihovny nebo Windows API používají callbacky. TCS je standardní bridge:

```csharp
public Task<string> StahniAsync(string url, CancellationToken ct)
{
    var tcs = new TaskCompletionSource<string>(TaskCreationOptions.RunContinuationsAsynchronously);

    // Registrace callbacku zrušení
    ct.Register(() => tcs.TrySetCanceled(ct));

    // Spuštění legacy operace s callbackem
    LegacyHttpClient.BeginGet(url,
        onSuccess: data  => tcs.TrySetResult(data),
        onError:   error => tcs.TrySetException(error));

    return tcs.Task;
}
```

### Jednorázová událost jako Task

Místo `ManualResetEvent` (blokující) lze použít TCS jako async signal:

```csharp
public class AsyncSignal
{
    private TaskCompletionSource _tcs = new(TaskCreationOptions.RunContinuationsAsynchronously);

    public Task CekejAsync() => _tcs.Task;

    public void Signalizuj() => _tcs.TrySetResult(); // TrySet* je bezpečné při vícenásobném volání

    public void Reset() => _tcs = new(TaskCreationOptions.RunContinuationsAsynchronously);
}

// Použití
var signal = new AsyncSignal();
_ = Task.Run(async () => { await Task.Delay(2000); signal.Signalizuj(); });
await signal.CekejAsync();
Console.WriteLine("Signál přijat!");
```

### Timeout wrapper pomocí TCS

```csharp
public async Task<T> STimeoutemAsync<T>(Task<T> task, TimeSpan timeout, CancellationToken ct = default)
{
    using var timeoutCts = CancellationTokenSource.CreateLinkedTokenSource(ct);
    timeoutCts.CancelAfter(timeout);

    var tcs = new TaskCompletionSource<T>(TaskCreationOptions.RunContinuationsAsynchronously);
    timeoutCts.Token.Register(() => tcs.TrySetCanceled(timeoutCts.Token));

    var prvni = await Task.WhenAny(task, tcs.Task);
    if (prvni == tcs.Task)
        throw new TimeoutException($"Operace nebyla dokončena do {timeout.TotalSeconds}s.");

    return await task; // propaguje výjimky původního tasku
}
```

### SetResult vs TrySetResult

```csharp
var tcs = new TaskCompletionSource<int>(TaskCreationOptions.RunContinuationsAsynchronously);

tcs.SetResult(42);           // Vyhodí InvalidOperationException při druhém volání
tcs.TrySetResult(42);        // Vrátí false – bezpečné při souběhu
tcs.TrySetException(ex);     // Bezpečné – nastaví pouze pokud task ještě neskončil
tcs.TrySetCanceled();        // Bezpečné
```

## 3. Kdy použít

- **Adaptace callback API** – starší Windows API, třetí strany (WebSocket events, SerialPort, FileSystemWatcher).
- **Jednorázové async signály** – čekání na inicializaci (startup gate), na připojení klienta, na první výsledek.
- **Bridging sync↔async hranice** – v místech, kde nelze přidat `async` do callchain (event handler, konstruktor).
- **Vlastní awaitable primitiva** – building block pro vlastní synchronizační primitiva (async semafory, turnstile).

**Kdy nepoužívat:** pokud lze metodu udělat `async` přímo – to je vždy jednodušší a méně náchylné k chybám. `Channel<T>` je lepší volba pro producent-konzument scénáře.

## 4. Časté chyby

- ❌ **Chybějící `RunContinuationsAsynchronously`** – pokračování awaiterů se spustí synchronně na vlákně volajícím `SetResult`, což může způsobit deadlock nebo blokování I/O vlákna.
- ❌ **`SetResult` místo `TrySetResult` při souběžném volání** – `SetResult` vyhodí `InvalidOperationException` pokud byl task již dokončen; `TrySet*` je bezpečnější v callbackech.
- ❌ **Zapomenutá registrace zrušení** – pokud `CancellationToken` není propagován do TCS, operace poběží i po zrušení; `ct.Register(() => tcs.TrySetCanceled(ct))` je standardní vzor.
- ❌ **TCS jako náhrada za Channel** – pro producent-konzument frontu použijte `Channel<T>`; TCS je pro jednorázové signály, ne pro stream hodnot.

---

<details>
<summary>Deep dive: TCS v SignalR, WebSockety a vlastní synchronizační primitiva</summary>

### TCS v SignalR – čekání na odpověď

SignalR interně používá TCS pro mapování invokací na odpovědi:

```csharp
// Zjednodušený princip InvocationRequest v SignalR
var pending = new ConcurrentDictionary<string, TaskCompletionSource<object>>();

public Task<object> InvokeAsync(string method, object[] args, CancellationToken ct)
{
    var id = Guid.NewGuid().ToString();
    var tcs = new TaskCompletionSource<object>(TaskCreationOptions.RunContinuationsAsynchronously);
    ct.Register(() => { pending.TryRemove(id, out _); tcs.TrySetCanceled(ct); });
    pending[id] = tcs;
    SendMessage(id, method, args); // odešle přes WebSocket
    return tcs.Task;
}

// Při příchodu odpovědi:
public void OnResponse(string id, object result)
{
    if (pending.TryRemove(id, out var tcs))
        tcs.TrySetResult(result);
}
```

### Vlastní AsyncLock pomocí TCS

```csharp
public class AsyncLock
{
    private readonly SemaphoreSlim _semaphore = new(1, 1);

    public async Task<IDisposable> LockAsync(CancellationToken ct = default)
    {
        await _semaphore.WaitAsync(ct);
        return new Releaser(_semaphore);
    }

    private sealed class Releaser(SemaphoreSlim s) : IDisposable
    {
        public void Dispose() => s.Release();
    }
}
```

### TCS vs Channel vs SemaphoreSlim

| Potřeba | Doporučení |
|---------|-----------|
| Jednorázový signal | `TaskCompletionSource` |
| Fronta zpráv | `Channel<T>` |
| Omezení souběžnosti | `SemaphoreSlim` |
| Vzájemné vyloučení | `AsyncLock` (postaveno na SemaphoreSlim) |

</details>

**Další krok:** [Interakce mezi asynchronním a synchronním kódem](Async_Sync_Interop.md)
