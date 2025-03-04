# Optimalizace práce s I/O (souborový systém, síťové operace)

I/O operace – čtení souborů, síťová komunikace, přístup k databázi – jsou ve většině aplikací největším zdrojem latence. CPU čeká na disk nebo síť, a pokud to neřešíte správně, zbytečně blokujete vlákna a snižujete škálovatelnost. Tato kapitola ukazuje, jak psát I/O kód, který neblokuje, efektivně bufferuje a zpracovává data jako proud.

## 1. Koncept

### Proč blokující I/O škodí

Každé vlákno .NET ThreadPoolu zabere přibližně 1 MB paměti zásobníku. Pokud máte 200 souběžných requestů a každý čeká na I/O blokujícím voláním, obsadíte 200 vláken – jen proto, aby čekala. Asynchronní I/O vrátí vlákno ThreadPoolu okamžitě po zahájení operace a přijme ho zpět teprve po dokončení. Výsledek: stejnou zátěž zvládnete s výrazně méně vlákny.

### Tři úrovně abstrakce pro I/O v .NET

| Abstrakce | Kdy použít |
|-----------|-----------|
| `StreamReader` / `StreamWriter` | Textové soubory, jednoduchá čtení |
| `FileStream` / `MemoryStream` | Binární data, přímá kontrola nad bufferem |
| `System.IO.Pipelines` (`PipeReader`/`PipeWriter`) | Vysoký throughput, parsování protokolů, síťové streamy |

## 2. Příklad

### Asynchronní čtení souboru – správně

```csharp
// ✅ Plně asynchronní, neblokuje vlákno
public async Task<string> CtiSouborAsync(string cesta, CancellationToken ct = default)
{
    // File.ReadAllTextAsync je pohodlné API pro malé soubory
    return await File.ReadAllTextAsync(cesta, ct);
}

// ✅ Pro velké soubory – čtení po blocích, konstantní paměťová stopa
public async Task ZpracujVelkySouborAsync(string cesta, CancellationToken ct = default)
{
    // FileOptions.Asynchronous je na Windows klíčové pro skutečně async I/O
    await using var stream = new FileStream(
        cesta, FileMode.Open, FileAccess.Read, FileShare.Read,
        bufferSize: 65536, options: FileOptions.Asynchronous | FileOptions.SequentialScan);

    var buffer = new byte[65536];
    int bytesRead;
    while ((bytesRead = await stream.ReadAsync(buffer, ct)) > 0)
    {
        ZpracujBlok(buffer.AsSpan(0, bytesRead));
    }
}
```

### Zápis souboru s bufferováním

```csharp
// ✅ BufferedStream snižuje počet syscallů – místo stovek malých zápisů provede jeden velký
public async Task ZapisSouborAsync(string cesta, IEnumerable<string> radky)
{
    await using var fs = new FileStream(cesta, FileMode.Create, FileAccess.Write,
        FileShare.None, bufferSize: 4096, useAsync: true);
    await using var bs = new BufferedStream(fs, bufferSize: 65536);
    await using var writer = new StreamWriter(bs);

    foreach (var radek in radky)
        await writer.WriteLineAsync(radek);
}
```

### Síťový přístup – HttpClient správně

```csharp
// ❌ Špatně – nová instance pro každý request vyčerpá sockety (TIME_WAIT)
public async Task<string> SpatneAsync(string url)
{
    using var client = new HttpClient(); // NIKDY takto
    return await client.GetStringAsync(url);
}

// ✅ Správně – IHttpClientFactory spravuje pooling socketů
public class StahovacSluzba(IHttpClientFactory factory)
{
    public async Task<string> StahniAsync(string url, CancellationToken ct = default)
    {
        using var client = factory.CreateClient();
        // ResponseHeadersRead = začni zpracovávat stream hned, nečekej na celé tělo
        using var response = await client.GetAsync(url, HttpCompletionOption.ResponseHeadersRead, ct);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadAsStringAsync(ct);
    }
}
```

### System.IO.Pipelines – pro vysoký throughput

```csharp
// PipeReader eliminuje kopírování bufferů – ideální pro parsování síťových protokolů
public async Task CtiPipeAsync(PipeReader reader, CancellationToken ct = default)
{
    while (true)
    {
        var result = await reader.ReadAsync(ct);
        var buffer = result.Buffer;

        while (TryParseZprava(ref buffer, out var zprava))
            await ZpracujZpravuAsync(zprava);

        reader.AdvanceTo(buffer.Start, buffer.End);

        if (result.IsCompleted) break;
    }
    await reader.CompleteAsync();
}
```

## 3. Kdy použít

- **`File.ReadAllTextAsync` / `WriteAllTextAsync`** – malé konfigurační soubory, jednoduché scénáře.
- **`FileStream` s `FileOptions.Asynchronous`** – velké soubory, binární data, přesná kontrola nad bufferem.
- **`BufferedStream`** – když zapisujete nebo čtete mnoho malých kusů dat; sdružuje je do větších I/O operací.
- **`IHttpClientFactory`** – všechny HTTP requesty v aplikaci; nikdy nevytvářejte `HttpClient` ručně v krátké životnosti.
- **`System.IO.Pipelines`** – síťové servery, parsování vlastních protokolů (HTTP, Redis, gRPC interně ho používají).

## 4. Časté chyby

- ❌ **Synchronní I/O ve webové aplikaci** – `File.ReadAllText()` (bez Async) blokuje vlákno ThreadPoolu a snižuje propustnost celého serveru.
- ❌ **`new HttpClient()` v každém requestu** – vyčerpá porty. Po ~20 000 requestech za hodinu začnete dostávat `SocketException`.
- ❌ **Příliš malý buffer na `FileStream`** – výchozí buffer je 4096 B. Pro sekvenční čtení velkých souborů použijte 64–256 KB.
- ❌ **Chybějící `FileOptions.Asynchronous` na Windows** – bez tohoto příznaku `FileStream` simuluje async přes blokující vlákno; skutečné async I/O nevyužijete.
- ❌ **Čtení celého souboru do paměti** – pro soubory > 10 MB používejte streaming; `ReadAllBytesAsync` načte vše do RAM najednou.

---

<details>
<summary>Deep dive: jak funguje async I/O uvnitř OS a kdy má smysl Pipelines</summary>

### Jak async I/O funguje na úrovni OS

Na Linuxu .NET používá `epoll` / `io_uring` (od .NET 6). Na Windows používá IOCP (I/O Completion Ports). Princip je stejný: operace se zaregistruje u OS, vlákno se vrátí do ThreadPoolu, a OS upozorní .NET runtime, když je I/O hotové. Runtime pak naplánuje pokračování na volné vlákno.

Klíčový rozdíl oproti simulovanému async (blokující vlákno na pozadí): skutečné async I/O nezabírá žádné vlákno po dobu čekání. Pro server zpracovávající 10 000 souběžných requestů je to rozdíl mezi desítkami a tisíci vláken.

### System.IO.Pipelines vs Stream

`Stream` má jeden zásadní problém: volající musí alokovat buffer předem a po přečtení dat ho zkopírovat. `PipeReader` funguje jinak – poskytne vám přímo pointer do svého interního bufferu (přes `ReadOnlySequence<byte>`). Nekopírujete data, jen je zparsujete in-place a pak řeknete `AdvanceTo`, kolik jste spotřebovali.

```
Stream: [allocate buffer] → [read into buffer] → [copy to parser] → výsledek
Pipe:   [reader.ReadAsync()] → [parser dostane slice interního bufferu] → [AdvanceTo] → výsledek
```

Pro vysokothroughputní server to znamená nula extra alokací na zpracování zprávy.

### Velikost bufferu – praktická pravidla

| Scénář | Doporučená velikost |
|--------|-------------------|
| Sekvenční čtení velkého souboru | 64–256 KB |
| Náhodný přístup k souboru | 4–16 KB |
| Síťový stream (API) | 8–32 KB |
| Síťový stream (file transfer) | 64–512 KB |

Příliš velký buffer není zadarmo – alokujete paměť, která se možná nevyužije. Příliš malý buffer zvyšuje počet syscallů. Správnou hodnotu vždy profilujte pro konkrétní workload.

</details>

**Další krok:** [Cache a vyrovnávací paměť](Cache.md)
