# CI/CD pro .NET (GitHub Actions, Azure DevOps)

Automatizované sestavení, testování a nasazení (CI/CD) zajišťuje, že každá změna v kódu je ověřena a bezpečně nasazena do cílového prostředí.

## 1. Koncept

**CI (Continuous Integration):** Při každém push/PR se spustí build a testy — rychlá zpětná vazba.
**CD (Continuous Deployment/Delivery):** Po úspěšném CI se artefakt nasadí do staging nebo produkce.

Doporučený postup:
1. Restore → Build → Test → Publish artefaktu
2. Nasazení do staging slotu / dev namespace
3. Smoke testy nebo manuální schválení
4. Swap / rollout do produkce

## 2. Příklad

### GitHub Actions — build a nasazení na Azure App Service

```yaml
# .github/workflows/dotnet.yml
name: CI/CD .NET

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: "8.0.x"

      - name: Restore
        run: dotnet restore

      - name: Build
        run: dotnet build --no-restore -c Release

      - name: Test
        run: dotnet test --no-build -c Release --logger "trx;LogFileName=results.trx"

      - name: Publish výsledků testů
        uses: dorny/test-reporter@v1
        if: always()
        with:
          name: .NET Tests
          path: "**/*.trx"
          reporter: dotnet-trx

      - name: Publish
        run: dotnet publish -c Release -o ./publish

      - name: Nasadit na Azure Web App
        uses: azure/webapps-deploy@v3
        with:
          app-name: webapp-mojekniha
          slot-name: staging
          publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE }}
          package: ./publish
```

### Azure DevOps Pipeline (azure-pipelines.yml)

```yaml
trigger:
  branches:
    include: [main]

pool:
  vmImage: ubuntu-latest

variables:
  buildConfiguration: Release

steps:
  - task: UseDotNet@2
    inputs:
      version: "8.0.x"

  - script: dotnet restore
    displayName: Restore

  - script: dotnet build --no-restore -c $(buildConfiguration)
    displayName: Build

  - task: DotNetCoreCLI@2
    displayName: Test
    inputs:
      command: test
      arguments: --no-build -c $(buildConfiguration) --collect "Code coverage"

  - task: DotNetCoreCLI@2
    displayName: Publish
    inputs:
      command: publish
      publishWebProjects: true
      arguments: -c $(buildConfiguration) -o $(Build.ArtifactStagingDirectory)

  - task: PublishBuildArtifacts@1
    inputs:
      pathToPublish: $(Build.ArtifactStagingDirectory)
```

## 3. Kdy použít

- **GitHub Actions** — projekty hostované na GitHubu; výborná integrace s GitHub Packages, Dependabot, OIDC pro Azure bez tajných hesel.
- **Azure DevOps** — enterprise prostředí s existující infrastrukturou DevOps, Boards, komplexní release gates.
- **OIDC místo publish profile** — pro GitHub Actions preferujte federated credentials (`azure/login` s OIDC) před publish profilem.

## 4. Časté chyby

- ❌ **Secrets přímo v YAML** — vždy ukládejte do GitHub Secrets nebo Azure DevOps Variable Groups; nikdy do repozitáře.
- ❌ **Vynechání `--no-restore` / `--no-build`** — redundantní kroky zpomalují pipeline; každý krok navazuje na předchozí.
- ❌ **Chybějící publikace výsledků testů** — pipeline "zelená" i při selhání testů, pokud se výsledky neanalyzují; vždy publikujte TRX/JUnit výstupy.
- ❌ **Nasazení přímo do produkce bez staging** — deployment slot swap umožňuje zero-downtime a rychlý rollback.

**Další krok:** [Observability – logy, metriky, tracing, OpenTelemetry](Observability.md)
