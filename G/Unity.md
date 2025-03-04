# Unity a C# – herní engine

Unity je nejrozšířenější herní engine pro 2D a 3D hry, mobilní i desktopové platformy. Skripty se píší v C#.

## Základní struktura skriptu

```csharp
using UnityEngine;

public class HracPohyb : MonoBehaviour
{
    public float rychlost = 5f;

    void Start()
    {
        // Zavolano jednou při spusštění objektu
        Debug.Log("Hrac se inicializoval.");
    }

    void Update()
    {
        // Zavolano každý snimek
        float x = Input.GetAxis("Horizontal");
        float z = Input.GetAxis("Vertical");

        Vector3 pohyb = new Vector3(x, 0, z) * rychlost * Time.deltaTime;
        transform.Translate(pohyb);
    }
}
```

- `MonoBehaviour` – základní třída všech Unity skriptů.
- `Start()` – inicializace (voláno jednou).
- `Update()` – každý snímek.
- `Time.deltaTime` – čas od posledního snímku (zajisc'ťuje nezávislost na FPS).

## Fyzika a kolize

```csharp
public class Projektil : MonoBehaviour
{
    void OnCollisionEnter(Collision collision)
    {
        if (collision.gameObject.CompareTag("Neprítel"))
        {
            // Zasáhl nepřítele
            Destroy(collision.gameObject);
            Destroy(gameObject);
        }
    }
}
```

## Hledání objektů a komponent

```csharp
// Nalezení jiného objektu
GameObject hrac = GameObject.FindWithTag("Player");
Rigidbody rb = hrac.GetComponent<Rigidbody>();
rb.AddForce(Vector3.up * 500f);

// Vytvoření objektu (Instantiate)
GameObject nepritele = Instantiate(prefabNepritele, spawnPoint.position, Quaternion.identity);
```

## UI v Unity

```csharp
using UnityEngine.UI;

public class HUDManager : MonoBehaviour
{
    public Text labelBody;  // Unity legacy UI
    // nebo: public TMP_Text labelBody; // TextMeshPro (doporučeno)

    void Update()
    {
        labelBody.text = $"Body: {GameManager.Body}";
    }
}
```

## Korutiny – async operace bez `async/await`

```csharp
IEnumerator OdpovedPoSekunde()
{
    yield return new WaitForSeconds(1f);
    Debug.Log("Uplynula 1 sekunda!");
}

// Spusštění
StartCoroutine(OdpovedPoSekunde());
```

---

<details>
<summary>Volitelné: Animace, ScriptableObjects a nové Input System</summary>

**Animace:**

```csharp
Animator anim = GetComponent<Animator>();
anim.SetBool("BehiVpred", true);
anim.SetTrigger("Skok");
```

**ScriptableObject** – data sdílená mezi objekty:

```csharp
[CreateAssetMenu(fileName = "ZbraneData", menuName = "Hra/Zbrane")]
public class ZbraneData : ScriptableObject
{
    public string Nazev;
    public float Poskozeni;
}
```

**Nový Input System:**

```csharp
using UnityEngine.InputSystem;

public void OnMove(InputAction.CallbackContext ctx)
{
    Vector2 smer = ctx.ReadValue<Vector2>();
}
```

</details>

---

**Další krok:** [MonoGame – alternativa k Unity](MonoGame.md)
