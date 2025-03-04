# Funkce a metody

Metoda je pojmenovaná část kódu, která provádí konkrétní úkol. Můžeme ji volat opakovaně.

## 1. Základní metoda

```csharp
void Pozdrav()
{
    Console.WriteLine("Ahoj!");
}

Pozdrav(); // volání metody
Pozdrav(); // lze volat vícekrát
```

- `void` – metoda nic nevrací.
- Metoda se spustí až když ji zavoláme.

## 2. Metoda s parametry

```csharp
void Pozdrav(string jmeno)
{
    Console.WriteLine($"Ahoj, {jmeno}!");
}

Pozdrav("Petr");   // Ahoj, Petr!
Pozdrav("Anna");   // Ahoj, Anna!
```

Parametr `jmeno` je vstupní hodnota, se kterou metoda pracuje.

## 3. Metoda s návratovou hodnotou

```csharp
int Sečti(int a, int b)
{
    return a + b;
}

int vysledek = Sečti(5, 3);
Console.WriteLine("Součet: " + vysledek); // Součet: 8
```

- Typ před názvem (`int`) určuje, co metoda vrací.
- `return` vrací hodnotu volájícímu kódu.

## 4. Lambda výraz (zkrácený zápis)

```csharp
int Sečti(int a, int b) => a + b;

Console.WriteLine(Sečti(4, 6)); // 10
```

Pro jednoduché metody lze tělo nahradit `=>` a výrazem.

---

<details>
<summary>Volitelné: Přetížení metod</summary>

Metody mohou mít stejný název, pokud mají jiné parametry:

```csharp
void Pozdrav() => Console.WriteLine("Ahoj, neznámý!");
void Pozdrav(string jmeno) => Console.WriteLine($"Ahoj, {jmeno}!");

Pozdrav();        // Ahoj, neznámý!
Pozdrav("Petr");  // Ahoj, Petr!
```

C# sám rozpozná, kterou variantu použít.

</details>

<details>
<summary>Volitelné: Rekurze</summary>

Metoda může volat sama sebe – nazývá se to rekurze:

```csharp
int Faktorial(int n)
{
    if (n <= 1) return 1;
    return n * Faktorial(n - 1);
}

Console.WriteLine(Faktorial(5)); // 120
```

Rekurze vyžaduje vždy **zastavítelný případ** (`if n <= 1`), jinak běží do nekonečna.

</details>

---

**Další krok:** [Pole a seznamy](Lists.md)
