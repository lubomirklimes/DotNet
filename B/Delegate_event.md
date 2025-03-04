# Delegáty a události (`delegate`, `event`)

Delegáty umožňují uložit **odkaz na metodu** a zavolat ji později. Události zasílají oznámení, že se něco stalo.

## 1. `Func<T>` a `Action<T>` – vestavěné delegáty

```csharp
// Action - metoda bez návr. hodnoty
Action<string> vypis = text => Console.WriteLine(text);
vypis("Ahoj!"); // Ahoj!

// Func - metoda s návr. hodnotou
Func<int, int, int> scitani = (a, b) => a + b;
Console.WriteLine(scitani(3, 5)); // 8
```

## 2. Vlastní `delegate`

```csharp
delegate int Operace(int a, int b);

int Scitani(int x, int y) => x + y;
int Nasobeni(int x, int y) => x * y;

Operace op = Scitani;
Console.WriteLine(op(3, 4)); // 7

op = Nasobeni;
Console.WriteLine(op(3, 4)); // 12
```

## 3. Multicast – více metod v jednom delegátu

```csharp
Action pozdrav = () => Console.WriteLine("Ahoj!");
pozdrav += () => Console.WriteLine("Hello!");

pozdrav(); // volá obě metody v pořadí
```

## 4. Události (`event`)

Událost je speciální deleggát – přihlášení pomocí `+=`, spouštění pomocí `Invoke`:

```csharp
class Budik
{
    public event Action Zazvoneni;

    public void Spust()
    {
        Console.WriteLine("Tik tak...");
        Zazvoneni?.Invoke(); // Spustí událost (pokud má nějakého odběratele)
    }
}

Budik b = new Budik();
b.Zazvoneni += () => Console.WriteLine("Vstavát!");
b.Spust();
// Tik tak...
// Vstavát!
```

## 5. `EventHandler<T>` – událost s daty

```csharp
class Stahovani
{
    public event EventHandler<int> Prubeh;

    public void Stahuj()
    {
        for (int i = 0; i <= 100; i += 25)
            Prubeh?.Invoke(this, i);
    }
}

var s = new Stahovani();
s.Prubeh += (sender, procenta) => Console.WriteLine($"Staženo: {procenta}%");
s.Stahuj();
```

---

<details>
<summary>Volitelné: Kdy použít co?</summary>

| | `Func`/`Action` | `delegate` | `event` |
|---|---|---|---|
| Předání metody | ano | ano | ne |
| Reakce na akce (GUI) | ne | ne | ano |
| Návr. hodnota | ano (`Func`) | ano | ne |

Pro většinu případů stačí `Func<T>` a `Action<T>`. Vlastní `delegate` a `event` používej pro GUI, senzory nebo vlastní frameworky.

</details>

---

**Další krok:** [Výrazy LINQ a zpracování kolekcí](LINQ.md)
