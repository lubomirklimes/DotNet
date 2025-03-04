# ASP.NET Core MVC

ASP.NET Core MVC je framework pro tvorbu webových aplikací založených na vzoru **Model – View – Controller**.

## Vytvoření projektu

```
dotnet new mvc -n MojeWebApp
```

## Struktura projektu

```
Controllers/   - logika požadavků
Views/         - Razor šablony (.cshtml)
Models/        - datové třídy
wwwroot/       - statické soubory (CSS, JS, obrázky)
Program.cs     - konfigurace
```

## Controller

```csharp
public class ProduktController : Controller
{
    private readonly IProduktService _service;

    public ProduktController(IProduktService service)
        => _service = service;

    // GET /Produkt
    public async Task<IActionResult> Index()
    {
        var produkty = await _service.GetVsechnyAsync();
        return View(produkty); // předá data do Views/Produkt/Index.cshtml
    }

    // GET /Produkt/Detail/5
    public async Task<IActionResult> Detail(int id)
    {
        var p = await _service.GetByIdAsync(id);
        if (p is null) return NotFound();
        return View(p);
    }

    // POST /Produkt/Vytvor
    [HttpPost]
    public async Task<IActionResult> Vytvor(Produkt model)
    {
        if (!ModelState.IsValid) return View(model);
        await _service.UlozAsync(model);
        return RedirectToAction(nameof(Index));
    }
}
```

## View (Razor)

```html
@model List<Produkt>

<h1>Produkty</h1>
<a asp-action="Vytvor">Přidat nový</a>

<table>
@foreach (var p in Model)
{
    <tr>
        <td>@p.Nazev</td>
        <td>@p.Cena.ToString("F2") Kč</td>
        <td><a asp-action="Detail" asp-route-id="@p.Id">Detail</a></td>
    </tr>
}
</table>
```

## Model a validace

```csharp
public class Produkt
{
    public int Id { get; set; }

    [Required(ErrorMessage = "Název je povinný.")]
    [MaxLength(100)]
    public string Nazev { get; set; }

    [Range(0.01, 99999)]
    public decimal Cena { get; set; }
}
```

## Routing

```csharp
// Atributový routing
[Route("api/produkty")]
public class ProduktApiController : Controller
{
    [HttpGet("{id:int}")]
    public IActionResult Get(int id) => Ok(/* ... */);
}
```

---

<details>
<summary>Volitelné: Middleware, filtry a oblasti (Areas)</summary>

**Middleware** zpracovává každý příchozí požadavek:

```csharp
app.Use(async (context, next) =>
{
    // před zpracováním
    await next();
    // po zpracování
});
```

**Filtry** (ActionFilter, AuthorizationFilter) umožňují opakování logiky (logování, autorizace).

**Areas** rozdělí velké projekty na pod-moduly (`/Admin`, `/Shop`, ...).

</details>

---

**Další krok:** [Razor Pages](RazorPages.md)
