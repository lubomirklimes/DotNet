# Objektově orientované programování (OOP)

OOP je způsob programování, kde si program rozdělíme na **objekty** s vlastnostmi a chováním.

## 1. Třída a objekt

**Třída** je šablona. **Objekt** je konkrétní věc vytvořená podle šablony.

```csharp
class Auto
{
    public string Znacka;
    public int RokVyroby;

    public void Start()
    {
        Console.WriteLine(Znacka + " startuje!");
    }
}

Auto mojeAuto = new Auto();
mojeAuto.Znacka = "Škoda";
mojeAuto.RokVyroby = 2022;
mojeAuto.Start(); // Škoda startuje!
```

## 2. Konstruktor

Konstruktor nastaví vlastnosti při vytvoření objektu:

```csharp
class Auto
{
    public string Znacka;
    public int RokVyroby;

    public Auto(string znacka, int rok)
    {
        Znacka = znacka;
        RokVyroby = rok;
    }
}

Auto auto = new Auto("Škoda", 2022);
Console.WriteLine($"{auto.Znacka}, {auto.RokVyroby}");
```

## 3. Vlastnosti (Properties)

Properties řídí přístup k datům třídy:

```csharp
class Osoba
{
    private int vek;

    public int Vek
    {
        get { return vek; }
        set
        {
            if (value >= 0) vek = value; // ochrana před záporným věkem
        }
    }
}
```

Zkrácený zápis pro jednoduché případy:

```csharp
class Osoba
{
    public string Jmeno { get; set; }
    public int Vek { get; private set; } // nastaví jen uvnitř třídy
}
```

## 4. Dědičnost

Třída může vycházet z jiné třídy a zdědit její vlastnosti i metody:

```csharp
class Zvire
{
    public string Jmeno;
    public void Zvuk() => Console.WriteLine(Jmeno + " vydává zvuk.");
}

class Pes : Zvire
{
    public void Stekej() => Console.WriteLine(Jmeno + " šteká!");
}

Pes rex = new Pes { Jmeno = "Rex" };
rex.Zvuk();   // Rex vydává zvuk.
rex.Stekej(); // Rex šteká!
```

---

<details>
<summary>Volitelné: Vizuální přepis metod (`override`) a rozhraní (`interface`)</summary>

**Override** – změní chování zděděné metody:

```csharp
class Zvire
{
    public virtual void Zvuk() => Console.WriteLine("Zvire vydává zvuk.");
}

class Pes : Zvire
{
    public override void Zvuk() => Console.WriteLine("Pes šteká!");
}

Zvire z = new Pes();
z.Zvuk(); // Pes šteká!
```

**Rozhraní (`interface`)** – definuje, jaké metody musí třída mít:

```csharp
interface IZvire
{
    void Zvuk();
}

class Kocka : IZvire
{
    public void Zvuk() => Console.WriteLine("Mnau!");
}
```

Rozhraní zajisc'žjí, že každá třída implementuje potřebné metody.

</details>

---

**Další krok:** [Výčtové typy (enum)](../B/Enum.md)
