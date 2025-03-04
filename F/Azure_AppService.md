# Azure App Service

Azure App Service je plně spravovaná PaaS platforma pro hostování webových aplikací, REST API a WebJobs bez nutnosti spravovat infrastrukturu.

## 1. Koncept

App Service spouští aplikaci v **App Service Plan** (definuje výpočetní prostředky, operační systém, cenu). Jedna plán může hostovat více aplikací.

Klíčové funkce:
- **Deployment slots** — staging/production s možností swap (blue-green deployment)
- **Auto-scaling** — horizontální škálování dle CPU, paměti nebo vlastní metriky
- **Managed Identity** — aplikace se autentizuje do Azure služeb bez connection stringů
- **Continuous deployment** — GitHub Actions, Azure DevOps, nebo přímý push přes ZIP

## 2. Příklad

### Nasazení přes Azure CLI

```bash
# Vytvoření Resource Group, App Service Plan a Web App
az group create -n rg-mojekniha -l westeurope
az appservice plan create -n plan-mojekniha -g rg-mojekniha --sku B2 --is-linux
az webapp create -n webapp-mojekniha -g rg-mojekniha \
  -p plan-mojekniha --runtime "DOTNET|8.0"

# Nasazení ZIP balíčku
dotnet publish -c Release -o ./publish
Compress-Archive ./publish/* ./app.zip
az webapp deployment source config-zip \
  -n webapp-mojekniha -g rg-mojekniha --src ./app.zip
```

### Deployment slot (blue-green)

```bash
az webapp deployment slot create \
  -n webapp-mojekniha -g rg-mojekniha --slot staging

# Nasadit do staging, otestovat, pak prohodit se production
az webapp deployment source config-zip \
  -n webapp-mojekniha -g rg-mojekniha --slot staging --src ./app.zip
az webapp deployment slot swap \
  -n webapp-mojekniha -g rg-mojekniha --slot staging
```

### Managed Identity + Key Vault v kódu

```csharp
// Program.cs — bez connection stringů nebo hesel v kódu
builder.Configuration.AddAzureKeyVault(
    new Uri($"https://{builder.Configuration["KeyVault:Name"]}.vault.azure.net/"),
    new DefaultAzureCredential()); // použije Managed Identity na App Service
```

### Konfigurace Application Settings

```bash
az webapp config appsettings set \
  -n webapp-mojekniha -g rg-mojekniha \
  --settings "KeyVault__Name=kv-mojekniha" "ASPNETCORE_ENVIRONMENT=Production"
```

## 3. Kdy použít

- **Webové aplikace a API** bez potřeby vlastní správy OS, patchování nebo kontejnerové orchestrace.
- **Rychlé nasazení** prototypů nebo menších aplikací.
- **Enterprise aplikace** s požadavkem na VNet integraci, Private Endpoint nebo App Service Environment (ASE).

Pro microservices s komplexní orchestrací zvažte **Azure Kubernetes Service** nebo **Azure Container Apps**.

## 4. Časté chyby

- ❌ **Connection stringy v appsettings.json** — vždy použijte App Service Application Settings nebo Key Vault; nastavení přepíše hodnoty v kódu.
- ❌ **Always On vypnuto u non-Free tier** — bez Always On se aplikace uspí a první request je pomalý (cold start).
- ❌ **Swap bez slot sticky settings** — citlivá nastavení (connection stringy, feature flags) označte jako "slot setting", aby se při swapu nepřenesla.
- ❌ **Scale-out bez session affinity** — pokud aplikace ukládá stav v paměti (session), multi-instance deployment selže; použijte Redis pro distribuovaný cache.

**Další krok:** [Azure Functions a Serverless vývoj](Azure_Functions.md)
