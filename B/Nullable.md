# Nullable reference types

Od C# 8 / .NET 6 jsou referenční typy ve výchozím stavu **non-nullable** – kompilátor varuje před možným `NullReferenceException` ještě před spuštěním kódu.

## 1. Koncept

Nullable reference types (NRT) jsou **statická analýza při kompilaci** – za běhu se nic nemění. Funkce se zapíná v `.csproj`:

```xml
<Nullable>enable</Nullable>
```

| Anotace | Význam |
|---|---|
| `string` | Nikdy null – kompilátor varuje, pokud může být null |
| `string?` | Může být null – kompilátor varuje, pokud nezkontrolujete |
| `string!` | Říkáte kompilátoru: "vím, že to null není" (null-forgiving) |

Kompilátor sleduje **flow analysis** – po `if (x != null)` ví, že `x` v tom bloku není null.

## 2. Příklad

### Základní použití

```csharp
// Bez nullable enable: string může být null a kompilátor to tiše přijme
// S nullable enable:
string  jmeno  = "Pavel";   // non-nullable – přiřazení null = varování
string? prijmeni = null;    // nullable – OK

// Přístup přes ?. a ?? – bezpečné
int delka = prijmeni?.Length ?? 0;

// Kontrola pro non-nullable přístup
if (prijmeni != null)
    Console.WriteLine(prijmeni.ToUpper()); // kompilátor ví: není null
```

### Null-forgiving operátor `!`

```csharp
// Použijte jen tehdy, kdy VY víte, že hodnota není null,
// ale kompilátor to nemůže dokázat (např. inicializace přes DI)
public class Sluzba
{
    private readonly ILogger<Sluzba> _logger = null!; // inicializováno přes DI
}
```

### Anotace parametrů a návratových typů

```csharp
// Metoda garantuje non-null výsledek
public string ZiskejJmeno(int id) { /* ... */ return "Pavel"; }

// Metoda může vrátit null
public string? NajdiJmeno(int id) { /* ... */ return null; }

// Pomocné atributy pro složitou analýzu
public bool TryGetValue(string key, [NotNullWhen(true)] out string? value)
{
    // Volající ví: pokud vrátí true, value != null
}
```

### Migrace existujícího kódu

```csharp
// 1. Zapněte <Nullable>enable</Nullable>
// 2. Opravujte varování postupně pomocí:
//    - přidání ? na nullable místa
//    - přidání null-checků
//    - přidání ! tam, kde jste si jisti
// Nikdy nevypínejte celý soubor – používejte granulární suprese:
#nullable disable warnings  // pro legacy sekce
```

## 3. Kdy použít

- **Všechny nové projekty** – zapněte `<Nullable>enable</Nullable>` od začátku; je mnohem snazší než migrace.
- **Veřejná API** – anotace `string?` vs `string` jsou součástí kontraktu; volající ví, co očekávat.
- **Migrace legacy kódu** – postupně, soubor po souboru; začněte od okrajových vrstev (modely, utility).

## 4. Časté chyby

- ❌ **`null!` jako zvyk** – null-forgiving operátor potlačuje ochranu; pokud ho používáte všude, ztrácíte výhodu NRT. Použijte pouze pro DI-inicializované pole nebo known-non-null situace.
- ❌ **Zapnutí nullable a ignorování varování** – varování jsou smysl funkce; build s `-warnaserror:nullable` varování vynucuje.
- ❌ **`string?` v interní logice, kde null nedává smysl** – pokud metoda nikdy nevrátí null, deklarujte `string`, ne `string?`; zbytečné null-checky u volajícího.
- ❌ **Spoléhání na NRT jako ochranu za běhu** – NRT jsou jen kompilátorová analýza. Null od externího zdroje (JSON, DB, uživatel) musíte stále validovat explicitně.

**Další krok:** [Pattern matching](PatternMatching.md)
