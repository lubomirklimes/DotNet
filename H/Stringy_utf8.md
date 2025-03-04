# Optimalizace stringů a práce s UTF-8 v .NET

`string` v .NET je UTF-16, immutable, heap-alokovaný. Každá operace vytvářející podřetězec (`Substring`, `Split`, string concatenation v cyklu) alokuje nový objekt. Pro high-performance kód existují lepší alternativy.

## 1. Koncept

### Kde vzniká problém

```
"Hello, World!".Substring(0, 5) → nový string "Hello" na heap (alokace + GC)
str1 + str2 + str3 v cyklu     → O(n²) alokací
```

### Nástroje

| Problém | Řešení |
|---------|--------|
| Concatenace v cyklu | `StringBuilder` |
| Slice bez alokace | `ReadOnlySpan<char>` |
| Tvorba stringu z Span | `string.Create()` |
| UTF-8 data (JSON, HTTP) | `ReadOnlySpan<byte>` + `Utf8JsonReader` |
| UTF-8 string literals | `"text"u8` (.NET 7+) |

→ Viz [Span_Memory.md](Span_Memory.md) pro obecné `Span<T>` a `Memory<T>`.

## 2. Příklad

### StringBuilder – concatenace v cyklu

```csharp
// ❌ O(n²) alokací – každý += vytvoří nový string
string result = "";
for (int i = 0; i < 1000; i++)
    result += i.ToString();

// ✅ StringBuilder – jedna alokace, amortizovaný append
var sb = new StringBuilder(capacity: 4096);
for (int i = 0; i < 1000; i++)
    sb.Append(i);
string result = sb.ToString();
```

### ReadOnlySpan\<char\> – slice bez alokace

```csharp
// ❌ Substring alokuje nový string
string first = "Hello, World!".Substring(0, 5); // alokace

// ✅ AsSpan() – žádná alokace, přímý pohled do původní paměti
ReadOnlySpan<char> first = "Hello, World!".AsSpan(0, 5);

// Parsování čísla bez alokace
ReadOnlySpan<char> input = "Answer: 42".AsSpan();
int colonIdx = input.IndexOf(':');
int value = int.Parse(input[(colonIdx + 2)..]);
```

### string.Create – tvorba stringu bez intermediate alokací

```csharp
// Sestavení stringu přímo do výsledného bufferu – žádná kopie
string guid = string.Create(36, Guid.NewGuid(), (span, g) =>
{
    g.TryFormat(span, out _);
});

// Rychlá tvorba base64-like ID
static string VytvorId(int length) =>
    string.Create(length, Random.Shared, (span, rng) =>
    {
        const string chars = "abcdefghijklmnopqrstuvwxyz0123456789";
        for (int i = 0; i < span.Length; i++)
            span[i] = chars[rng.Next(chars.Length)];
    });
```

### UTF-8 string literály (.NET 7+)

```csharp
// "text"u8 → ReadOnlySpan<byte> v UTF-8 – statická paměť, žádná alokace
ReadOnlySpan<byte> hello = "Hello, World!"u8;
ReadOnlySpan<byte> json  = "{\"status\":\"ok\"}"u8;

// Přímý zápis UTF-8 do streamu bez konverze
await response.Body.WriteAsync(hello, ct);
```

### Utf8JsonWriter – JSON bez string alokací

```csharp
// Psaní JSON přímo do PipeWriter nebo ArrayBufferWriter
var writer = new ArrayBufferWriter<byte>();
using var json = new Utf8JsonWriter(writer);
json.WriteStartObject();
json.WriteString("name"u8, uzivatel.Jmeno);
json.WriteNumber("age"u8, uzivatel.Vek);
json.WriteEndObject();
json.Flush();

await response.Body.WriteAsync(writer.WrittenMemory, ct);
```

### Interpolované stringy a string.Format – kdy jsou OK

```csharp
// .NET 6+: interpolované stringy jsou optimalizovány kompilátorem
// → nepoužívají string.Format interně při jednoduchých případech
string msg = $"Uživatel {jmeno} ({vek} let)"; // OK pro ne-hot-path

// Pro hot path (logování, metriky) použijte podmíněné formátování:
if (log.IsEnabled(LogLevel.Debug))
    log.LogDebug("Zpracováno {Count} položek", count);
```

## 3. Kdy použít

- **`ReadOnlySpan<char>`** – parsování protokolů (HTTP headers, CSV, konfigurace), kdy každý request parsuje stejný formát.
- **`StringBuilder`** – dynamická tvorba HTML, SQL, report textů kde počet operací není předem znám.
- **`string.Create()`** – tvorba stringu s pevnou délkou (ID, hash hex-string, formatovaný timestamp).
- **`"text"u8` a `Utf8JsonWriter`** – JSON API endpoints, gRPC, kde payload je UTF-8 a nechcete konvertovat UTF-16 → UTF-8.

## 4. Časté chyby

- ❌ **`string +=` v cyklu** – klasická O(n²) chyba; vždy `StringBuilder` pro více než ~5 concatenací.
- ❌ **`string.Split` na hot path** – alokuje pole a nové stringy pro každý prvek; použijte `MemoryExtensions.Split` nebo ruční parsování přes `Span`.
- ❌ **`ToString()` na `ReadOnlySpan<char>`** – okamžitě alokuje nový string; zbytečné, pokud jen potřebujete předat do metody přijímající `ReadOnlySpan<char>`.
- ❌ **Interpolovaný string v logování** – `log.LogDebug($"Data: {expensiveCall()}")` vyhodnotí `expensiveCall()` i když Debug log je vypnutý. Vždy strukturované logování: `log.LogDebug("Data: {Value}", expensiveCall())`.
- ❌ **`Utf8String` API** – `Utf8String` byl experimentální typ; v produkčním kódu používejte `ReadOnlySpan<byte>` a `"..."u8` literály.

---

<details>
<summary>Deep dive: String interning, CompositeFormat a výkonové benchmarky</summary>

### String interning

.NET automaticky internuje string literály (string pool). `string.Intern(s)` lze použít pro dynamicky vzniklé stringy, které se opakují – ale jen zřídka (interned stringy nikdy GC neuvolní).

```csharp
// Vhodné: konstantní klíče HTTP hlaviček
string header = string.Intern(rawHeader); // opakující se "Content-Type", "Authorization"
```

### CompositeFormat (.NET 8+)

Pre-parsovaný formátovací vzor – eliminuje opakované parsování format stringu:

```csharp
private static readonly CompositeFormat _format =
    CompositeFormat.Parse("Uživatel {0} provedl akci {1} v {2}");

string msg = string.Format(CultureInfo.InvariantCulture, _format, user, action, time);
```

### Benchmarky (BenchmarkDotNet, .NET 8)

| Operace | Čas | Alokace |
|---------|-----|---------|
| `"Hello".Substring(0, 3)` | 20 ns | 32 B |
| `"Hello".AsSpan(0, 3)` | 1 ns | 0 B |
| `int.Parse("42")` | 15 ns | 32 B |
| `int.Parse("42".AsSpan())` | 8 ns | 0 B |
| `"a" + "b" + "c"` × 1000 | 1.2 ms | 4 MB |
| `StringBuilder` × 1000 | 0.05 ms | 8 KB |

</details>

**Další krok:** [ref struct a readonly struct](Ref_Readonly_Struct.md)
