# Azure Functions a Serverless vývoj

Azure Functions je **serverless** platforma – píšeš pouze kód funkce, Azure se stará o infrastrukturu.

## Vytvoření projektu

```bash
dotnet new func --worker-runtime dotnet-isolated -n MojeFunkce
```

## HTTP Trigger – nejběžnější trigger

```csharp
public class PozdravFunction
{
    [Function("Pozdrav")]
    public IActionResult Run(
        [HttpTrigger(AuthorizationLevel.Function, "get", Route = "pozdrav/{jmeno}")] HttpRequest req,
        string jmeno)
    {
        return new OkObjectResult($"Ahoj, {jmeno}!");
    }
}
```

## Timer Trigger – naplnované spuštění

```csharp
public class DenniReport
{
    [Function("DenniReport")]
    public async Task Run(
        [TimerTrigger("0 0 6 * * *")] TimerInfo timer) // Každý den v 6:00
    {
        // Zpracování
    }
}
```

CRON formát: `{sekunda} {minuta} {hodina} {den} {měsíc} {den_týdne}`

## Blob Trigger – reakce na soubor

```csharp
public class ZpracujObrazek
{
    [Function("ZpracujObrazek")]
    public async Task Run(
        [BlobTrigger("obrazky/{name}", Connection = "AzureStorage")] Stream blob,
        string name)
    {
        // Zpracování obrazku ze Storage
    }
}
```

## Typy triggerů

| Trigger | Spuštění |
|---|---|
| HTTP | Příchozí HTTP požadavek |
| Timer | CRON plán |
| Blob | Nový/změněný soubor v Azure Storage |
| Queue | Zpráva v Azure Queue |
| Service Bus | Zpráva v Azure Service Bus |
| Event Grid | Událost z jakéhokoli Azure zdroje |

## Lokální vývoj a nasazení

```bash
# Spuštění lokálně
func start

# Nasazení do Azure
az functionapp create --name MojeFunkce --consumption-plan-location westeurope ...
func azure functionapp publish MojeFunkce
```

---

<details>
<summary>Volitelné: Durable Functions a orchestrace</summary>

**Durable Functions** umožňují stavové workflowy:

```csharp
[Function("Orchestrator")]
public async Task<string> RunOrchestrator(
    [OrchestrationTrigger] TaskOrchestrationContext context)
{
    var krok1 = await context.CallActivityAsync<string>("Krok1", null);
    var krok2 = await context.CallActivityAsync<string>("Krok2", krok1);
    return krok2;
}
```

Patterny: **Function Chaining**, **Fan-out/Fan-in**, **Human Interaction**.

</details>

---

**Další krok:** [Práce s databázemi v cloudu (SQL, NoSQL)](Cloud_Databases.md)
