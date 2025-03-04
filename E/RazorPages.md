# Razor Pages

Razor Pages je stránkově orientovaný model pro server-side web v ASP.NET Core. Každá stránka má svůj `.cshtml` soubor a párový **Page Model** (`.cshtml.cs`) – handler metody pro GET, POST atd.

## 1. Koncept

Na rozdíl od MVC (Controller + View odděleny) jsou v Razor Pages logika a šablona fyzicky u sebe. Vhodné pro aplikace orientované na stránky (formuláře, CRUD, admin rozhraní) bez složité routing logiky.

```
Pages/
  Products/
    Index.cshtml        ← šablona
    Index.cshtml.cs     ← PageModel (handler)
    Edit.cshtml
    Edit.cshtml.cs
```

## 2. Příklad

### PageModel – handler metody

```csharp
// Pages/Products/Index.cshtml.cs
public class IndexModel(IProduktService sluzba) : PageModel
{
    public IReadOnlyList<Produkt> Produkty { get; private set; } = [];

    // GET /Products
    public async Task OnGetAsync(CancellationToken ct)
    {
        Produkty = await sluzba.GetAllAsync(ct);
    }
}
```

### Šablona (.cshtml)

```html
@page
@model IndexModel
<h1>Produkty</h1>
<ul>
@foreach (var p in Model.Produkty)
{
    <li><a asp-page="./Edit" asp-route-id="@p.Id">@p.Nazev</a> – @p.Cena Kč</li>
}
</ul>
<a asp-page="./Create">Přidat produkt</a>
```

### POST – formulář s model bindingem

```csharp
public class CreateModel(IProduktService sluzba) : PageModel
{
    [BindProperty]
    public ProduktDto Input { get; set; } = new();

    public IActionResult OnGet() => Page();

    public async Task<IActionResult> OnPostAsync(CancellationToken ct)
    {
        if (!ModelState.IsValid)
            return Page();

        await sluzba.CreateAsync(Input, ct);
        return RedirectToPage("./Index");
    }
}
```

### Registrace v Program.cs

```csharp
builder.Services.AddRazorPages();
app.MapRazorPages();
```

## 3. Kdy použít

- **Admin rozhraní, CRUD aplikace** – formuláře s GET + POST jsou přirozené v page modelu.
- **Jednoduché webové stránky** – kontaktní formuláře, landing pages, dokumentační weby.
- **Přechod z Web Forms** – stránkový model je bližší Web Forms než MVC.

Preferujte MVC nebo Minimal API pokud potřebujete složité routing, API endpointy, nebo testovatelné controllery nezávislé na HTTP kontextu.

## 4. Časté chyby

- ❌ **Chybějící `@page` direktiva** – bez `@page` je soubor obyčejný Razor view, ne Razor Page; routing nefunguje.
- ❌ **`[BindProperty]` bez `SupportsGet = true` pro GET parametry** – výchozí `[BindProperty]` binduje jen POST; pro query string přidejte `[BindProperty(SupportsGet = true)]`.
- ❌ **Business logika přímo v PageModel** – PageModel je controller; logika patří do service tříd injektovaných přes DI.
- ❌ **Absence `ModelState.IsValid` check po POST** – bez validace se nevalidní data dostanou do databáze.

**Další krok:** [Blazor – moderní webová aplikace v C#](Blazor.md)
