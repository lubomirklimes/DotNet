# Anonymní typy a `Tuple`

Anonymní typy a Tuple umožňují **seskupit více hodnot** bez nutnosti definovat novou třídu.

## 1. Anonymní typy

```csharp
var osoba = new { Jmeno = "Petr", Vek = 30 };
Console.WriteLine($"{osoba.Jmeno}, {osoba.Vek} let");
```

- Hodnoty **nelze měnit** po vytvoření.
- Nelze vrátit z metody – používají se hlavně v LINQ.

## 2. `ValueTuple` – více hodnot z metody

```csharp
(string Jmeno, int Vek) ZiskejOsobu()
{
    return ("Petr", 30);
}

var o = ZiskejOsobu();
Console.WriteLine($"{o.Jmeno}, {o.Vek}");
```

Nebo s dekonstrukcí:

```csharp
var (jmeno, vek) = ZiskejOsobu();
Console.WriteLine($"{jmeno}, {vek}");

// Nepotřebnou hodnotu ignorujeme _:
var (_, vek2) = ZiskejOsobu();
```

## 3. Kdy co použít

| | Anonymní typ | ValueTuple | class / record |
|---|---|---|---|
| Vrací z metody | ne | ano | ano |
| Pojmenované položky | ano | ano | ano |
| Složitá logika | ne | ne | ano |
| Typické použití | LINQ select | více návr. hodnot | datové modely |

---

<details>
<summary>Volitelné: Starší `Tuple` (referenční typ)</summary>

```csharp
var t = Tuple.Create("Petr", 30);
Console.WriteLine(t.Item1); // Petr
Console.WriteLine(t.Item2); // 30
```

Starší `Tuple` používá `Item1`, `Item2`... což je méně čitelné. `ValueTuple` s pojmenovanými položkami je modernější a dopormučená varianta.

</details>

---

**Další krok:** [Delegáty a události (delegate, event)](Delegate_event.md)
