# Docker a Kubernetes v .NET

## Docker

Docker zabalová aplikaci a její závislosti do **kontejneru** – behí stejně kdekoli.

### Dockerfile pro .NET aplikaci

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:9.0
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "MojeApp.dll"]
```

```bash
docker build -t moje-app .
docker run -p 8080:80 moje-app
```

### Visual Studio integrace

Visual Studio přidá Dockerfile automaticky: **Pravé tlačítko na projekt -> Přidat -> Podpora Dockeru**.

### Docker Compose (více služeb)

```yaml
# docker-compose.yml
services:
  api:
    build: ./Api
    ports:
      - "8080:80"
    environment:
      - ConnectionStrings__Default=Server=db;Database=MojeDB
    depends_on:
      - db

  db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=Heslo123!
```

```bash
docker compose up -d
```

## Kubernetes

Kubernetes (K8s) spravuje kontejnery ve více uzlech.

### Základní pojmy

| Pojem | Popis |
|---|---|
| **Pod** | Nejmenší jednotka – jeden nebo více kontejnerů |
| **Deployment** | Spravuje repliky Podů |
| **Service** | Zprostředkovává síťový přístup k Podům |
| **ConfigMap / Secret** | Konfigurace a tajemství |

### Deployment manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: moje-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: moje-api
  template:
    metadata:
      labels:
        app: moje-api
    spec:
      containers:
        - name: moje-api
          image: moje-api:latest
          ports:
            - containerPort: 80
          env:
            - name: ASPNETCORE_ENVIRONMENT
              value: Production
```

```bash
kubectl apply -f deployment.yaml
kubectl get pods
kubectl logs -l app=moje-api
```

---

<details>
<summary>Volitelné: Health checks a Helm</summary>

**.NET health checks** (pro Kubernetes liveness/readiness probe):

```csharp
builder.Services.AddHealthChecks()
    .AddDbContextCheck<AppDbContext>()
    .AddUrlGroup(new Uri("https://api.example.com"), "externí API");

app.MapHealthChecks("/health");
```

**Helm** je správce balíčků pro Kubernetes – umožňuje parametrizované nasazení.

</details>

---

**Další krok:** [Azure App Service](Azure_AppService.md)
