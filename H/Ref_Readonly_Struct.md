# `ref struct` a `readonly struct` – výkonové struktury v .NET

`struct` v .NET je hodnotový typ – předává se kopií. `ref struct` a `readonly struct` jsou rozšíření, která přinášejí **výkonové záruky a omezení** potřebná pro zero-allocation kód.

## 1. Koncept

### readonly struct

Označení celé struktury jako `readonly` říká kompilátoru, že žádné pole nesmí být po konstruktoru změněno. Kompilátor pak může:
- eliminovat **obranné kopie** při volání metod přes `in` parametry
- přesněji optimalizovat předávání hodnotou

### ref struct

`ref struct` je struktura, která smí existovat **pouze na zásobníku** (stack). Nemůže být:
- uložena jako člen třídy nebo jiného non-ref struct
- použita v `async` metodě, lambda, iterátoru
- boxována do `object` nebo rozhraní

Kompilátor tato pravidla hlídá. Primární příklad: `Span<T>` a `ReadOnlySpan<T>` jsou `ref struct`.

→ Viz [Span_Memory.md](Span_Memory.md) – `Span<T>` jako hlavní `ref struct` v .NET.

## 2. Příklad

### readonly struct – immutabilní hodnotový typ

```csharp
public readonly struct Vector2
{
    public float X { get; }
    public float Y { get; }

    public Vector2(float x, float y) => (X, Y) = (x, y);

    public float Length => MathF.Sqrt(X * X + Y * Y);

    // Metody smí být volány přes 'in' bez obranné kopie
    public Vector2 Normalized()
    {
        float len = Length;
        return new Vector2(X / len, Y / len);
    }
}

// in parametr: předá referenci bez kopírování (readonly struct = bezpečné)
static float VypocitejUhel(in Vector2 a, in Vector2 b)
    => MathF.Acos(Vector2.Dot(a.Normalized(), b.Normalized()));
```

### ref struct – stack-only typ

```csharp
public ref struct StackBuffer
{
    private Span<byte> _data;
    private int _position;

    public StackBuffer(Span<byte> buffer)
    {
        _data = buffer;
        _position = 0;
    }

    public void Write(ReadOnlySpan<byte> bytes)
    {
        bytes.CopyTo(_data[_position..]);
        _position += bytes.Length;
    }

    public ReadOnlySpan<byte> WrittenSpan => _data[.._position];
}

// Použití na zásobníku – žádná heap alokace
Span<byte> stackMem = stackalloc byte[512];
var buf = new StackBuffer(stackMem);
buf.Write("Hello"u8);
buf.Write(", World!"u8);
SendResponse(buf.WrittenSpan);
```

### Obranné kopie – proč readonly struct záleží

```csharp
public struct Mutable
{
    public int Value;
    public int GetValue() => Value; // non-readonly metoda
}

// Při předání přes 'in': kompilátor vytvoří obrannou kopii pro každé volání metody
// → readonly struct tento problém eliminuje
static void Test(in Mutable m)
{
    _ = m.GetValue(); // kopie #1
    _ = m.GetValue(); // kopie #2 (kompilátor neví, zda metoda mění stav)
}
```

### readonly ref struct (C# 11+)

```csharp
// Kombinace: stack-only + immutabilní
public readonly ref struct ReadOnlyBuffer
{
    private readonly ReadOnlySpan<byte> _data;
    public ReadOnlyBuffer(ReadOnlySpan<byte> data) => _data = data;
    public ReadOnlySpan<byte> Data => _data;
}
```

## 3. Kdy použít

- **`readonly struct`** – immutabilní hodnotové typy: vektory, body, barvy, měny, souřadnice. Kdekoli chcete zaručit, že hodnota se nezmění a předávání přes `in` je časté.
- **`ref struct`** – stack-only kontejnery pro práci s `Span<T>`, parsery, temporary buffery. Kdekoli nechcete heap alokaci a životnost je omezena na jednu metodu/call stack.
- **`readonly ref struct`** – immutabilní pohled na stack-allocated data; pohled na paměť bez možnosti modifikace.

## 4. Časté chyby

- ❌ **`ref struct` jako člen třídy** – kompilátorská chyba; `ref struct` je stack-only. Použijte `Memory<T>` nebo normální `struct`.
- ❌ **`ref struct` v `async` nebo iterator metodě** – kompilátorská chyba; state machine async metody je třída na heapu, nemůže držet `ref struct`.
- ❌ **`struct` bez `readonly` s `in` parametrem** – kompilátor vytváří obranné kopie pro každé volání metody, čímž se eliminuje výkonová výhoda `in`.
- ❌ **Mutable struct předaná hodnotou** – `struct` se kopíruje; modifikace kopie neovlivní originál. Záludné při `foreach` a LINQ.
- ❌ **Velká struct předávaná hodnotou** – structs > ~16 B jsou dražší na kopírování než reference. Preferujte `in` parametry nebo zvažte třídu.

---

<details>
<summary>Deep dive: in/out/ref parametry, stack depth a C# 13 novinky</summary>

### in vs ref vs out

```csharp
static void Test(
    in  Vector2 a,   // read-only reference – volající vidí referenci, metoda nesmí zapisovat
    ref Vector2 b,   // read-write reference – metoda může číst i zapisovat
    out Vector2 c)   // write-only – metoda musí přiřadit před návratem
{
    c = new Vector2(a.X + b.X, a.Y + b.Y);
    b = c; // povoleno
}
```

### Escape analysis a stack depth

Kompilátor zaručí, že `ref struct` neopustí zásobníkový rámec, kde vznikl. Při předání `ref struct` do jiné metody jako `ref` / `in` parametr je zachována záruky.

### C# 13 – ref struct v generikách a rozhraních

```csharp
// C# 13 umožňuje omezená použití ref struct v generikách
// (experimentální, vyžaduje allows ref struct constraint)
static void Zpracuj<T>(T data) where T : allows ref struct { /* ... */ }
```

### Kdy struct vs class

| Kritérium | struct | class |
|-----------|--------|-------|
| Velikost | < 16 B ideálně | Libovolná |
| Životnost | Krátká (zásobník nebo pole) | Libovolná |
| Identita | Hodnotová (kopíruje se) | Referenční (sdílí se) |
| Dědičnost | Ne (pouze rozhraní) | Ano |
| GC tlak | Nulový (stack) | Ano (heap) |

</details>

**Další krok:** [Minimalizace přepínání vláken a ThreadPool tuning](Prepinani_vlaken.md)
