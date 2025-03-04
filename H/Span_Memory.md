# `Span<T>` a `Memory<T>` – zero-copy práce s pamětí

`Span<T>` a `Memory<T>` umožňují pracovat s **libovolným blokem paměti** (pole, řetězec, stack, nativní paměť) bez kopírování. Jsou základem pro high-performance kód v ASP.NET Core, System.IO.Pipelines a moderních .NET API.

## 1. Koncept

### Span\<T\>

`Span<T>` je `ref struct` – existuje **pouze na zásobníku** (stack). Umí odkazovat na:
- segment existujícího pole (`array[2..7]`)
- stackalloc paměť
- nativní (unmanaged) paměť

Protože je na zásobníku, nemůže být uložen jako člen třídy, předán do asynchronní metody, ani boxován. Kompilátor to hlídá.

### Memory\<T\>

`Memory<T>` je strukturální ekvivalent pro **heap** – může být uložen jako člen třídy, předán do async metody, použit ve `Task`. Z `Memory<T>` se kdykoli získá `Span<T>` přes `.Span`.

### ReadOnly varianty

`ReadOnlySpan<T>` a `ReadOnlyMemory<T>` zakazují zápis. `string` se mapuje na `ReadOnlySpan<char>` – zásadní optimalizace pro zpracování textu bez alokace.

> Viz [Stringy_utf8.md](Stringy_utf8.md) pro `ReadOnlySpan<char>` a string optimalizace.  
> Viz [Ref_Readonly_Struct.md](Ref_Readonly_Struct.md) pro `ref struct` omezení a `readonly struct`.  
> Viz [ArrayPool.md](ArrayPool.md) pro pooling bufferů kombinovaný se `Span<T>`.

## 2. Příklad

### Span\<T\> – slice bez kopírování

```csharp
int[] numbers = { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };

// žádná alokace – Span odkazuje na existující paměť
Span<int> slice = numbers.AsSpan(2, 5); // prvky 3..7
foreach (int n in slice)
    Console.Write($"{n} "); // 3 4 5 6 7

// Zápis přes Span modifikuje původní pole
slice[0] = 99;
Console.WriteLine(numbers[2]); // 99
```

### ReadOnlySpan\<char\> – práce s řetězcem bez alokace

```csharp
ReadOnlySpan<char> text = "Hello, World!".AsSpan();
ReadOnlySpan<char> hello = text[..5]; // "Hello" – žádný nový string

// Parsování čísla z podřetězce bez alokace
ReadOnlySpan<char> numStr = "Price: 42 CZK".AsSpan(7, 2);
int price = int.Parse(numStr); // 42
```

### stackalloc + Span

```csharp
// Alokace na zásobníku – ultra-rychlé, automaticky uvolněné při návratu z metody
// Bezpečné jen pro malé buffery (< ~1 KB)
Span<byte> buffer = stackalloc byte[256];
int bytesWritten = Encoding.UTF8.GetBytes("Hello", buffer);
Process(buffer[..bytesWritten]);
```

### Memory\<T\> v asynchronní metodě

```csharp
// Span nelze použít v async metodě – Memory ano
public async Task ZpracujAsync(Memory<byte> buffer, CancellationToken ct)
{
    int read = await _stream.ReadAsync(buffer, ct);

    // Získáme Span pro synchronní zpracování přečtených dat
    Span<byte> data = buffer.Span[..read];
    ParseData(data);
}
```

### Parsování protokolu bez alokace

```csharp
// HTTP request line: "GET /index.html HTTP/1.1"
ReadOnlySpan<char> line = requestLine.AsSpan();
int firstSpace  = line.IndexOf(' ');
int secondSpace = line[firstSpace..].IndexOf(' ') + firstSpace;

ReadOnlySpan<char> method  = line[..firstSpace];          // "GET"
ReadOnlySpan<char> path    = line[(firstSpace+1)..secondSpace]; // "/index.html"
ReadOnlySpan<char> version = line[(secondSpace+1)..];     // "HTTP/1.1"
// žádná string alokace – přímá práce s původní pamětí
```

## 3. Kdy použít

- **Parsování textových protokolů** (HTTP, CSV, JSON) bez intermediárních string alokací.
- **Práce s binárními buffery** – serializace, deserializace, kryptografie, komprese.
- **Slicing polí** – zpracování části dat bez `Array.Copy` nebo `new[]`.
- **Rozhraní s nativním kódem** – `MemoryMarshal`, `MemoryHandle` pro P/Invoke bez kopírování.
- **System.IO.Pipelines** – `PipeReader.ReadAsync` vrací `ReadOnlySequence<byte>` pracující přes `Span`.

**Nepoužívejte** `Span<T>` v async metodách – použijte `Memory<T>`. Pro dlouhodobé buffery kombinujte `Memory<T>` s `ArrayPool<T>`.

## 4. Časté chyby

- ⚠ **`Span<T>` jako člen třídy** – kompilátor.ská chyba; `ref struct` nelze uložit na heap. Použijte `Memory<T>`.
- ⚠ **`Span<T>` v `async` metodě** – kompilátor.ská chyba; span nemůže přežít `await` point. Použijte `Memory<T>`.
- ⚠ **stackalloc bez kontroly velikosti** – stack overflow při velkém bufferu (> ~1 KB). Pravidlo: stackalloc jen pro malé, pevně dané velikosti.
- ⚠ **Slice za konec** – `span[..n]` kde `n > span.Length` vyhodí `ArgumentOutOfRangeException`. Vždy ověřte délku.
- ⚠ **`Span` na stackalloc paměti vrácené z metody** – pointer na zásobník je neplatný po návratu z metody. Kompilátor.ská chyba u safe kódu, nebezpečí v `unsafe`.

---

<details>
<summary>Deep dive: MemoryMarshal, System.IO.Pipelines a výkonové benchmarky</summary>

### MemoryMarshal – pokročilá reinterpretace

```csharp
// Reinterpretace byte[] jako Span<int> – žádná kopie, žádná alokace
byte[] bytes = new byte[16];
Span<int> ints = MemoryMarshal.Cast<byte, int>(bytes); // 4 prvky int
ints[0] = 0x12345678;
// bytes[0..3] nyní obsahuje 0x78, 0x56, 0x34, 0x12 (little-endian)
```

### System.IO.Pipelines

`PipeReader` a `PipeWriter` jsou postaveny na `Span<T>`/`Memory<T>` pro zero-copy I/O:

```csharp
public async Task CtiPipeAsync(PipeReader reader, CancellationToken ct)
{
    while (true)
    {
        ReadResult result = await reader.ReadAsync(ct);
        ReadOnlySequence<byte> buffer = result.Buffer;

        // Zpracování bez kopírování
        foreach (ReadOnlyMemory<byte> segment in buffer)
            Process(segment.Span);

        reader.AdvanceTo(buffer.End);
        if (result.IsCompleted) break;
    }
}
```

### Benchmarky (BenchmarkDotNet)

| Operace | Tradiční (`string.Substring`) | `ReadOnlySpan<char>` |
|---------|------------------------------|----------------------|
| Slice 10× | 150 ns, 80 B alokace | 5 ns, 0 B |
| Parse int z podřetězce | 200 ns, 40 B | 15 ns, 0 B |
| Kopírování 1 KB pole | 300 ns, 1024 B | 50 ns, 0 B |

</details>

**Další krok:** [Optimalizace alokace paměti pomocí ArrayPool<T>](ArrayPool.md)