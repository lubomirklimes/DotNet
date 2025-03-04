# Fyzika a kolize ve hrách

## Fyzika v Unity

Unity má vestavěný fyzikální engine (NVIDIA PhysX). Fyzika funguje přes komponentu `Rigidbody`.

### `Rigidbody` – fyzikální těleso

```csharp
public class GranatomethProject : MonoBehaviour
{
    void Start()
    {
        Rigidbody rb = GetComponent<Rigidbody>();
        rb.AddForce(transform.forward * 1000f, ForceMode.Impulse);
        rb.useGravity = true;
    }
}
```

### Kolize

```csharp
void OnCollisionEnter(Collision collision)
{
    Debug.Log($"Narazil jsem do: {collision.gameObject.name}");
    // Získání bodu dopadu
    ContactPoint kontakt = collision.contacts[0];
    Debug.Log($"Bod dopadu: {kontakt.point}");
}

// Trigger (bez fyzické reakce)
void OnTriggerEnter(Collider other)
{
    if (other.CompareTag("Powerup"))
        NasbierejPowerup(other.gameObject);
}
```

### Raycast – detekce objektu pod kurzorem / v směru

```csharp
void Update()
{
    Ray paprsek = Camera.main.ScreenPointToRay(Input.mousePosition);
    if (Physics.Raycast(paprsek, out RaycastHit zasah, maxVzdalenost: 100f))
    {
        Debug.Log($"Zasahl jsem: {zasah.collider.gameObject.name}");
        Debug.DrawLine(paprsek.origin, zasah.point, Color.red);
    }
}
```

## Fyzika v MonoGame (ruční implementace)

MonoGame nemá vestavěnou fyziku – buď implementuji sám, nebo použiji knihovnu **Aether.Physics2D**:

```bash
dotnet add package Aether.Physics2D.MG
```

```csharp
using tainicom.Aether.Physics2D.Dynamics;

World svet = new World(new Vector2(0, -9.8f)); // gravitace

Body telo = svet.CreateBody(pozice, 0f, BodyType.Dynamic);
telo.CreateCircle(0.5f, 1f); // poloměr, hustota

// V Update
svet.Step((float)gameTime.ElapsedGameTime.TotalSeconds);
Vector2 poziceObjektu = telo.Position;
```

## Základní fyzikální koncepty ve hrách

| Pojem | Popis |
|---|---|
| **Rigidbody** | Objekt s fyzikálním chováním (hmotnost, síly) |
| **Collider** | Tvar pro detekci kolizí (krabice, kruh, mesh) |
| **Trigger** | Collider bez fyzické reakce |
| **Raycast** | Vysílá paprsek a vrací zásah |
| **Layer mask** | Filtruje, které vrstvy raycasta zasahují |

---

**Další krok:** [CLI nástroje v .NET](CLI.md)
