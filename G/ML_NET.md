# ML.NET

ML.NET je open-source framework Microsoftu pro strojové učení v C# a F# bez nutnosti Pythonu. Umožňuje trénovat a nasazovat modely přímo v .NET ekosystému.

## 1. Koncept

ML.NET pracuje s konceptem **pipeline** — řetěz transformací dat (načtení, předzpracování, trénink, hodnocení). Výsledkem je `ITransformer`, který lze serializovat a opakovaně používat pro inference.

Typy úloh:
| Úloha | Příklad |
|-------|---------|
| Binární klasifikace | Spam/ne-spam, podvod/ne-podvod |
| Vícehodnotová klasifikace | Kategorizace textu |
| Regrese | Předpověď ceny, počtu |
| Clusterování | Segmentace zákazníků |
| Doporučování | Doporučení produktů |
| Detekce anomálií | Výpadky, podvody v časových řadách |

**Model Builder** (Visual Studio extension) a `mlnet` CLI umožňují automatické hledání nejlepšího algoritmu (AutoML).

## 2. Příklad

### Binární klasifikace — predikce sentimentu

```csharp
// dotnet add package Microsoft.ML
using Microsoft.ML;
using Microsoft.ML.Data;

public class RecenzeSentiment
{
    [LoadColumn(0)] public string Text { get; set; } = "";
    [LoadColumn(1), ColumnName("Label")] public bool JePositivni { get; set; }
}

public class PredikceVysledku
{
    [ColumnName("PredictedLabel")] public bool JePositivni { get; set; }
    public float Pravdepodobnost { get; set; }
}

var mlContext = new MLContext(seed: 42);

// Načtení dat
var data = mlContext.Data.LoadFromTextFile<RecenzeSentiment>(
    "recenze.tsv", hasHeader: true, separatorChar: '\t');

var rozdeleni = mlContext.Data.TrainTestSplit(data, testFraction: 0.2);

// Pipeline
var pipeline = mlContext.Transforms.Text
    .FeaturizeText("Features", nameof(RecenzeSentiment.Text))
    .Append(mlContext.BinaryClassification.Trainers.SdcaLogisticRegression());

// Trénink
var model = pipeline.Fit(rozdeleni.TrainSet);

// Hodnocení
var metriky = mlContext.BinaryClassification.Evaluate(
    model.Transform(rozdeleni.TestSet));
Console.WriteLine($"Přesnost: {metriky.Accuracy:P2}");

// Uložení modelu
mlContext.Model.Save(model, data.Schema, "sentiment.zip");
```

### Inference (predikce v produkci)

```csharp
var mlContext = new MLContext();
var model = mlContext.Model.Load("sentiment.zip", out _);
var engine = mlContext.Model
    .CreatePredictionEngine<RecenzeSentiment, PredikceVysledku>(model);

var vysledek = engine.Predict(new RecenzeSentiment { Text = "Výborný produkt!" });
Console.WriteLine($"Pozitivní: {vysledek.JePositivni}, P={vysledek.Pravdepodobnost:P1}");
```

### AutoML přes CLI

```bash
dotnet tool install -g mlnet
mlnet classification --dataset recenze.tsv --label-col JePositivni --train-time 30
```

## 3. Kdy použít

- **Klasifikace textu nebo dat** — kategorizace tiketů, detekce podvodů, predikce odchodu zákazníků.
- **Inference v .NET aplikaci** — pokud nechcete volat Python mikroservice; ML.NET modely běží in-process.
- **ONNX modely** — ML.NET umí načíst ONNX model (trénovaný v PyTorch/TF) a volat jej z C#.

Pro LLM a generativní AI viz Microsoft Foundry / Azure OpenAI SDK; ML.NET je určen pro klasické ML úlohy.

## 4. Časté chyby

- ❌ **`PredictionEngine` není thread-safe** — pro produkci použijte `PredictionEnginePool<TIn,TOut>` z `Microsoft.Extensions.ML`.
- ❌ **Trénink při každém spuštění aplikace** — model trénujte offline, uložte jako `.zip` a v produkci jen načítejte.
- ❌ **Malý nebo nevyvážený dataset** — ML.NET model bude přeučený nebo bude předpovídat jen majoritní třídu; zkontrolujte distribuci a použijte oversampling nebo váhování.
- ❌ **Ignorování metrik hodnocení** — `Accuracy` může být klamná u nevyvážených dat; sledujte `F1Score`, `AUC` a `ConfusionMatrix`.

**Další krok:** [Techniky pro profilování aplikací .NET](../H/Profilovani_aplikaci.md)
