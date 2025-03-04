# Azure Key Vault

Azure Key Vault je spravovaná služba pro bezpečné ukládání **tajných hodnot** (connection stringy, API klíče), **kryptografických klíčů** a **certifikátů** mimo zdrojový kód.

## 1. Koncept

Tajné hodnoty jsou verzované, auditované (každý přístup je logován v Activity Log) a přístupné přes RBAC nebo Access Policy. Aplikace se autentizuje pomocí **Managed Identity** — bez hesel nebo connection stringů v kódu.

Integrace s ASP.NET Core probíhá přes `AddAzureKeyVault` Configuration Provider — tajné hodnoty se načtou jako součást `IConfiguration` a aplikace je používá stejně jako lokální `appsettings.json`.

Pojmenování tajných hodnot: Key Vault nepovoluje `:`; používejte `--` jako oddělovač sekcí (např. `ConnectionStrings--Default` → `ConnectionStrings:Default` v .NET).

## 2. Příklad

### Načtení tajných hodnot přes Configuration Provider

```csharp
// dotnet add package Azure.Extensions.AspNetCore.Configuration.Secrets
// dotnet add package Azure.Identity
using Azure.Identity;

var builder = WebApplication.CreateBuilder(args);

if (!builder.Environment.IsDevelopment())
{
    builder.Configuration.AddAzureKeyVault(
        new Uri($"https://{builder.Configuration["KeyVault:Name"]}.vault.azure.net/"),
        new DefaultAzureCredential());
}

// Tajné hodnoty dostupné přes IConfiguration jako obvykle:
// builder.Configuration["ConnectionStrings:Default"]
```

### Ruční přístup k tajné hodnotě

```csharp
using Azure.Security.KeyVault.Secrets;

var client = new SecretClient(
    new Uri("https://mujvault.vault.azure.net/"),
    new DefaultAzureCredential());

KeyVaultSecret secret = await client.GetSecretAsync("DbPassword");
Console.WriteLine(secret.Value);

// Uložení nové tajné hodnoty
await client.SetSecretAsync("NoveHeslo", "super-tajne-123");
```

### Přiřazení RBAC role přes CLI

```bash
# Managed Identity aplikace musí mít roli "Key Vault Secrets User"
az keyvault create -n mujvault -g rg-moje -l westeurope
az webapp identity assign -n webapp-moje -g rg-moje
PRINCIPAL=$(az webapp identity show -n webapp-moje -g rg-moje --query principalId -o tsv)
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee $PRINCIPAL \
  --scope $(az keyvault show -n mujvault -g rg-moje --query id -o tsv)
```

### Certifikát z Key Vault pro HTTPS

```csharp
// Azure App Service může certifikát načíst přímo přes UI nebo CLI:
// az webapp config ssl import --key-vault mujvault --key-vault-certificate-name MujCert ...
```

## 3. Kdy použít

- **Vždy pro produkční tajné hodnoty** — connection stringy, API klíče třetích stran, OAuth secrets.
- **Rotace klíčů** — Key Vault podporuje automatickou rotaci; aplikace vždy čte aktuální verzi.
- **Certifikáty** — centrální správa TLS certifikátů sdílených více aplikacemi.

## 4. Časté chyby

- ❌ **Tajné hodnoty v `appsettings.json` nebo v kódu** — git history je trvalá; jedinou bezpečnou alternativou v cloudu je Key Vault.
- ❌ **Pojmenování s `:` místo `--`** — Key Vault odmítne `ConnectionStrings:Default`; použijte `ConnectionStrings--Default`.
- ❌ **Přístup přes Access Policy místo RBAC** — Microsoft doporučuje RBAC jako primární model; Access Policy bude v budoucnu deprecated.
- ❌ **Bez cache** — každý `GetSecretAsync` je HTTP call; Configuration Provider cachuje hodnoty při startu. Pro runtime rotation použijte `SecretClient` s vlastní cache (např. `IMemoryCache` s expiry).

**Další krok:** [CI/CD pro .NET aplikace](CICD.md)
