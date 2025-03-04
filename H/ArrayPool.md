# Optimalizace alokace paměti pomocí `ArrayPool<T>`

`ArrayPool<T>` umožňuje **znovupoužívat pole** místo jejich opakované alokace. Eliminuje tlak na Garbage Collector při práci s dočasnými buffery – typicky při I/O, serializaci, síťové komunikaci.

## 1. Koncept

Každá alokace `new byte[n]` vytvoří objekt na haldě. GC ho musí sledovat a uvolnit. Při stovkách requestů za sekundu to způsobuje časté Gen 0/Gen 2 sbírky.

`ArrayPool<T>.Shared` udržuje interní fond polí. `Rent(minLength)` vrátí pole ze skladu (nebo alokuje nové), `Return(array)` ho vrátí zpět. Pole může být větší, než jste žádali – vždy pracujte s přesnou délkou, ne s `array.Length`.

**Kritické pravidlo:** Vrácené pole **není vynulované**. Pokud odesíláte data nebo pracujete s citlivými informacemi, vynulujte nebo předejte `clearArray: true` do `Return`.

→ Viz [Sprava_pameti.md](Sprava_pameti.md) pro širší kontext GC a LOH.  
→ Viz [Span_Memory.md](Span_Memory.md) pro kombinaci `ArrayPool` se `Span<T>`.

## 2. Příklad

### Základní vzor – Rent / Return

```csharp
var pool = ArrayPool<byte>.Shared;
byte[] buffer = pool.Rent(4096); // minimální velikost; dostanete >= 4096 B

try
{
    int read = await stream.ReadAsync(buffer.AsMemory(0, 4096), ct);
    Process(buffer.AsSpan(0, read)); // pracujte jen s přečtenými bajty!
}
finally
{
    pool.Return(buffer); // VŽDY v finally – i při výjimce
}
```

### Kombinace s IDisposable pro elegantní API

```csharp
// Zapůjčení ve using – automatické vrácení
public static IDisposable Zapujc(int minSize, out byte[] buffer)
{
    buffer = ArrayPool<byte>.Shared.Rent(minSize);
    var captured = buffer;
    return new Vracec(() => ArrayPool<byte>.Shared.Return(captured));
}

private sealed class Vracec(Action akce) : IDisposable { public void Dispose() => akce(); }

// Použití
using (Zapujc(8192, out var buf))
{
    // buf je platné v tomto bloku
}
```

### Vynulování citlivých dat

```csharp
byte[] buffer = ArrayPool<byte>.Shared.Rent(1024);
try
{
    NaplnHeslem(buffer);
    OdesliSecure(buffer);
}
finally
{
    // clearArray: true vynuluje pole před vrácením do poolu
    ArrayPool<byte>.Shared.Return(buffer, clearArray: true);
}
```

### Vlastní pool pro specifické požadavky

```csharp
// Vlastní pool s omezením – pro izolaci od sdíleného Shared poolu
private static readonly ArrayPool<byte> _lokalniPool =
    ArrayPool<byte>.Create(maxArrayLength: 64 * 1024, maxArraysPerBucket: 20);

public async Task ZpracujAsync(CancellationToken ct)
{
    var buffer = _lokalniPool.Rent(16 * 1024);
    try { /* ... */ }
    finally { _lokalniPool.Return(buffer); }
}
```

### MemoryPool\<T\> – pro System.IO.Pipelines

```csharp
// MemoryPool<T> vrací IMemoryOwner<T> – automatické Return při Dispose
using IMemoryOwner<byte> owner = MemoryPool<byte>.Shared.Rent(4096);
Memory<byte> memory = owner.Memory;
int read = await stream.ReadAsync(memory, ct);
Parse(memory.Span[..read]);
// owner.Dispose() vrátí paměť do poolu
```

## 3. Kdy použít

- **Čtení a zápis streamů** – HTTP requesty, file I/O, WebSocket, SerialPort.
- **Serializace/deserializace** – intermediate buffery pro JSON, Protobuf, MessagePack.
- **Síťové zpracování** – parsování paketů, TCP/UDP komunikace.
- **Pole > 85 KB** – tato jdou automaticky na LOH; pooling zabrání fragmentaci LOH.
- **Hot path s vysokou frekvencí** – stovky volání za sekundu, kde GC tlak je měřitelný.

**Nepoužívejte** pro malá pole (< 128 B) nebo jednorázové alokace – režie poolu nepřináší zisk.

## 4. Časté chyby

- ❌ **Práce s `array.Length` místo skutečné délky** – `Rent(100)` může vrátit pole délky 128; přečtení dat mimo skutečnou délku = garbage nebo přetečení.
- ❌ **Chybějící `Return` při výjimce** – pole zůstane mimo pool; pool alokuje nové a paměť roste. Vždy `try/finally`.
- ❌ **Použití pole po `Return`** – pool pole znovu přidělí jinému volajícímu; race condition a poškozená data.
- ❌ **Citlivá data bez `clearArray: true`** – hesla, tokeny, osobní data zůstanou v poolovaném poli a mohou uniknout dalšímu volajícímu.
- ❌ **`new ArrayPool<T>()`** – `ArrayPool<T>` je abstraktní; instanciujte jen přes `ArrayPool<T>.Shared` nebo `ArrayPool<T>.Create(...)`.

---

<details>
<summary>Deep dive: Interní implementace, benchmarky a System.Buffers</summary>

### Jak funguje ArrayPool\<T\>.Shared

`Shared` je instance `TlsOverPerCoreLockedStacksArrayPool<T>`:
- Každé CPU jádro má **thread-local sklad** polí per bucket (velikostní třída: 16, 32, 64, ... B).
- `Rent` nejprve zkusí thread-local sklad (bez zamykání), pak sdílený zásobník pro dané jádro.
- `Return` vrátí pole do thread-local skladu; přebytečná pole jsou zahozena (GC je uvolní normálně).

### Benchmarky (BenchmarkDotNet, .NET 8, 10K iterací)

| Operace | `new byte[4096]` | `ArrayPool.Rent(4096)` |
|---------|-----------------|----------------------|
| Čas | 250 ns | 15 ns |
| Alokace | 4096 B | 0 B |
| GC Gen 0 / 1000 op | 2–4 | 0 |

### System.Buffers jmenný prostor

`System.Buffers` obsahuje:
- `ArrayPool<T>` – pooling polí
- `MemoryPool<T>` – pooling s `IMemoryOwner<T>` (automatický Return)
- `ReadOnlySequence<T>` – pro non-contiguous buffery (System.IO.Pipelines)
- `BufferWriter<T>` – pro přímý zápis do poolovaného bufferu

```csharp
// Buffers kombinace v ASP.NET Core middleware:
var writer = new ArrayBufferWriter<byte>();
JsonSerializer.Serialize(writer, data);
await response.Body.WriteAsync(writer.WrittenMemory, ct);
```

</details>

**Další krok:** [Optimalizace stringů a práce s UTF-8](Stringy_utf8.md)
