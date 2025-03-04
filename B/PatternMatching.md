# Pattern matching

Pattern matching v C# umožňuje **testovat tvar a hodnotu dat** a zároveň je extrahovat – v jednom výrazu. Od C# 7 se postupně rozšiřuje; C# 9–11 přidaly `switch` výrazy, relační a logické vzory.

## 1. Koncept

Pattern matching nahrazuje řetězce `if/else` s přetypováním čitelnějším a bezpečnějším kódem. Kompilátor kontroluje **exhaustiveness** u `switch` výrazů – chybějící větev je varování nebo chyba.

Hlavní typy vzorů:

| Vzor | Příklad | Co dělá |
|---|---|---|
| Type pattern | `x is string s` | Test + extrakce do `s` |
| Property pattern | `x is { Jmeno: "Pavel" }` | Test vlastností |
| Tuple pattern | `(x, y) is (0, 0)` | Test n-tice |
| Relational | `x is > 0 and < 100` | Porovnání hodnot |
| Logical | `x is null or ""` | Kombinace vzorů |
| List pattern | `arr is [1, 2, ..]` | Test pole/seznamu (C# 11) |

## 2. Příklad

### is výraz s type pattern

```csharp
object obj = "hello";

// Starý způsob
if (obj is string)
{
    string s = (string)obj;
    Console.WriteLine(s.Length);
}

// ✅ Pattern matching – test + extrakce v jednom
if (obj is string s)
    Console.WriteLine(s.Length); // s je string v tomto bloku
```

### Switch výraz (C# 8+)

```csharp
// Funkce vrací popis tvaru – exhaustive switch
string PopisTvaru(object tvar) => tvar switch
{
    Circle  { Polomer: > 0 } c => $"Kruh r={c.Polomer}",
    Rectangle r when r.Sirka == r.Vyska => $"Čtverec {r.Sirka}x{r.Vyska}",
    Rectangle r => $"Obdélník {r.Sirka}x{r.Vyska}",
    null        => "null",
    _           => "Neznámý tvar"   // discard – default větev
};
```

### Property pattern a relační vzory

```csharp
// Property pattern – testuje vlastnosti objektu
bool JeValidniUzivatel(Uzivatel u) => u is
{
    Jmeno: { Length: > 0 },
    Vek:   >= 18,
    Email: not null
};

// Relační + logické vzory
string KategorizujTeplotu(double t) => t switch
{
    < 0        => "Mráz",
    >= 0 and < 20  => "Zima",
    >= 20 and < 30 => "Příjemné",
    >= 30      => "Horko",
    _          => "Neznámé"
};
```

### Tuple pattern pro stavový automat

```csharp
// Přechody stavu (aktuální stav, vstup) -> nový stav
Stav DalsiStav(Stav aktualni, Vstup vstup) => (aktualni, vstup) switch
{
    (Stav.Neaktivni, Vstup.Start) => Stav.Bezici,
    (Stav.Bezici,    Vstup.Pause) => Stav.Pozastaveno,
    (Stav.Pozastaveno, Vstup.Start) => Stav.Bezici,
    (_, Vstup.Stop) => Stav.Neaktivni,
    _ => aktualni // ostatní kombinace nemění stav
};
```

### List pattern (C# 11)

```csharp
bool JePrazdny(int[] arr)   => arr is [];
bool MaJedenPrvek(int[] arr) => arr is [_];
bool ZacinaNulou(int[] arr)  => arr is [0, ..];
bool MaTriPrvky(int[] arr)   => arr is [_, _, _];
```

## 3. Kdy použít

- **Discriminated union simulace** – zpracování různých podtypů jednoho rozhraní (Result, Error, Success).
- **Stavové automaty** – tuple pattern pro (stav, vstup) → nový stav.
- **Validace dat** – property pattern místo vnořených `if` pro komplexní podmínky.
- **Deserializace variant** – JSON s různými typy zpráv zpracovaný přes `switch` na property vzorech.

## 4. Časté chyby

- ❌ **Chybějící `_` větev v `switch` výrazu** – runtime `MatchFailureException` pokud žádný vzor nesedí. Vždy přidejte fallback nebo zajistěte exhaustiveness.
- ❌ **Pořadí vzorů záleží** – konkrétnější vzory musí být před obecnějšími; `Rectangle { Sirka: > 0 }` musí předcházet obecnému `Rectangle`.
- ❌ **Property pattern na null** – `x is { Jmeno: "Pavel" }` vrátí `false` pokud `x == null`; není nutný explicitní null-check před vzorem.
- ❌ **Přílišná komplexita** – pattern matching je čitelný pro 3–5 větví; při 10+ větví zvažte tabulku dispatch nebo polymorfismus.

**Další krok:** [Extension methods](ExtensionMethods.md)
