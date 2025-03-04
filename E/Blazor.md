# Blazor – webové aplikace v C#

Blazor umožňuje psát interaktivní webové aplikace v C# místo JavaScriptu.

## Dva režimy Blazor

| Režim | Kde běží C# | Vhodné pro |
|---|---|---|
| **Blazor Server** | Server, UI aktualizace přes SignalR | Intranetové aplikace |
| **Blazor WebAssembly** | Prohlížeč (Wasm) | Offline, klientské aplikace |
| **Blazor United** (.NET 8+) | Kombinace | Moderní full-stack |

## Vytvoření projektu

```
dotnet new blazor -n MojeBlazorApp
```

## Komponenta

```razor
@* Counter.razor *@
<h3>Počítadlo: @pocet</h3>
<button @onclick="Inkrementuj">+1</button>

@code {
    private int pocet = 0;
    private void Inkrementuj() => pocet++;
}
```

## Parametry komponenty

```razor
@* Pozdrav.razor *@
<p>Ahoj, @Jmeno!</p>

@code {
    [Parameter] public string Jmeno { get; set; } = "";
}
```

```razor
@* Použití *@
<Pozdrav Jmeno="Petr"/>
```

## Volaíní API a async

```razor
@inject HttpClient Http

@if (produkty is null)
{
    <p>Načítám...</p>
}
else
{
    @foreach (var p in produkty)
    {
        <p>@p.Nazev - @p.Cena Kč</p>
    }
}

@code {
    private List<Produkt>? produkty;

    protected override async Task OnInitializedAsync()
    {
        produkty = await Http.GetFromJsonAsync<List<Produkt>>("api/produkty");
    }
}
```

## Formuláře a validace

```razor
<EditForm Model="@model" OnValidSubmit="Odeslat">
    <DataAnnotationsValidator/>
    <ValidationSummary/>

    <InputText @bind-Value="model.Jmeno" placeholder="Jméno"/>
    <ValidationMessage For="@(() => model.Jmeno)"/>

    <button type="submit">Uložit</button>
</EditForm>

@code {
    private OsobaModel model = new();
    private void Odeslat() { /* uložit data */ }
}
```

---

<details>
<summary>Volitelné: JavaScript interop a kaskády událostí</summary>

Blazor může volat JavaScript a naopak:

```csharp
@inject IJSRuntime JS

await JS.InvokeVoidAsync("console.log", "Ahoj z C#!");
string vysledek = await JS.InvokeAsync<string>("prompt", "Zadej jméno:");
```

Události mezi komponentami: použij `EventCallback` nebo sdílený službní singleton (`Scoped`/`Singleton` služba + `StateHasChanged()`).

</details>

---

**Další krok:** [Minimal APIs](MinimalAPI.md)
