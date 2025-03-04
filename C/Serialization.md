# Serializace a deserializace

Serializace převádí objekt na text/binární data, deserializace naopak.

## JSON pomocí `System.Text.Json`

```csharp
using System.Text.Json;

record Osoba(string Jmeno, int Vek);

// Serializace (objekt -> JSON)
var osoba = new Osoba("Petr", 30);
string json = JsonSerializer.Serialize(osoba);
Console.WriteLine(json);
// {"Jmeno":"Petr","Vek":30}

// Deserializace (JSON -> objekt)
Osoba? nactena = JsonSerializer.Deserialize<Osoba>(json);
Console.WriteLine(nactena?.Jmeno); // Petr
```

## Nastavení formátování

```csharp
var moznosti = new JsonSerializerOptions
{
    WriteIndented = true,                    // hezké odsazení
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase // camelCase klíče
};

string json = JsonSerializer.Serialize(osoba, moznosti);
// {
//   "jmeno": "Petr",
//   "vek": 30
// }
```

## Čtení JSON souboru

```csharp
string json = await File.ReadAllTextAsync("data.json");
var seznam = JsonSerializer.Deserialize<List<Osoba>>(json);
```

## Atributy pro mapování

```csharp
using System.Text.Json.Serialization;

class Produkt
{
    [JsonPropertyName("product_name")]
    public string Nazev { get; set; }

    [JsonIgnore]
    public string InterniBod { get; set; }  // nebude v JSON
}
```

---

<details>
<summary>Volitelné: `Newtonsoft.Json` a vlastní konvertóry</summary>

`Newtonsoft.Json` (NuGet: `Newtonsoft.Json`) je starší, ale stále velmi používaná knihovna:

```csharp
using Newtonsoft.Json;

string json = JsonConvert.SerializeObject(osoba, Formatting.Indented);
Osoba? o = JsonConvert.DeserializeObject<Osoba>(json);
```

Pro vlastní konverzi typů implementuj `JsonConverter<T>` z `System.Text.Json.Serialization`.

</details>

---

**Další krok:** [Dependency Injection a práce se službami](DependencyInjection.md)
