# Middleware a pipeline v ASP.NET Core

ASP.NET Core zpracovává každý HTTP request přes **pipeline middleware** – řetěz komponent, kde každá může request zpracovat, upravit nebo předat dál.

## 1. Koncept

Pipeline je uspořádaná sekvence middleware. Každý middleware:
1. Přijme request (může ho upravit)
2. Předá dál (`await next(context)`)
3. Zpracuje response (může ho upravit při návratu)

```
Request → [Middleware 1] → [Middleware 2] → [Middleware N] → Endpoint
Response ← [Middleware 1] ← [Middleware 2] ← [Middleware N] ←
```

Pořadí registrace **záleží** – middleware registrovaný dříve obaluje pozdější.

## 2. Příklad

### Standardní pořadí v ASP.NET Core

```csharp
// Program.cs – pořadí middleware je pevně dané konvencí
app.UseExceptionHandler("/Error");   // musí být první pro zachytávání výjimek
app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();
app.UseAuthentication();             // kdo jsi?
app.UseAuthorization();              // smíš to udělat?
app.MapControllers();                // endpointy
```

### Vlastní middleware – třída

```csharp
public class RequestTimingMiddleware(RequestDelegate next, ILogger<RequestTimingMiddleware> log)
{
    public async Task InvokeAsync(HttpContext context)
    {
        var sw = Stopwatch.StartNew();
        try
        {
            await next(context); // předání dál
        }
        finally
        {
            sw.Stop();
            log.LogInformation(
                "{Method} {Path} -> {Status} za {Ms}ms",
                context.Request.Method,
                context.Request.Path,
                context.Response.StatusCode,
                sw.ElapsedMilliseconds);
        }
    }
}

// Registrace
app.UseMiddleware<RequestTimingMiddleware>();
```

### Inline middleware pomocí Use/Map/Run

```csharp
// Use – zpracuje a předá dál
app.Use(async (context, next) =>
{
    context.Response.Headers.Append("X-App-Version", "1.0");
    await next(context);
});

// Map – rozvětví pipeline pro danou cestu
app.Map("/healthz", branch =>
    branch.Run(ctx => ctx.Response.WriteAsync("OK")));

// Run – terminální middleware, nepředává dál
app.Run(ctx => ctx.Response.WriteAsync("404 - Not Found"));
```

### Short-circuit (přerušení pipeline)

```csharp
// Middleware, který nepředá dál při splnění podmínky
app.Use(async (context, next) =>
{
    if (!context.Request.Headers.ContainsKey("X-Api-Key"))
    {
        context.Response.StatusCode = 401;
        await context.Response.WriteAsync("API key required");
        return; // nepředáváme dál
    }
    await next(context);
});
```

## 3. Kdy použít

- **Cross-cutting concerns** – logování, timing, korelační ID, hlavičky bezpečnosti, rate limiting.
- **Vlastní autentizace** – middleware pro validaci vlastních tokenů před `UseAuthentication`.
- **Request/Response transformace** – komprese, body rewriting, caching.

Preferujte **endpoint filtry** (`IEndpointFilter`) nebo **action filtry** v MVC pro logiku specifickou pro jeden endpoint místo globálního middleware.

## 4. Časté chyby

- ❌ **Špatné pořadí middleware** – `UseAuthentication` musí být před `UseAuthorization`; `UseRouting` před `UseAuthentication` v starším kódu. Porušení způsobí tiché selhání autorizace.
- ❌ **Výjimka v middleware bez `try/finally`** – middleware musí ošetřit výjimky nebo je propagovat; jinak response zůstane nekompletní.
- ❌ **Blokující kód v middleware** – middleware je volán pro každý request; synchronní blokování degraduje propustnost. Vždy async.
- ❌ **`Run` místo `Use` uprostřed pipeline** – `Run` je terminální, nepředá request dál; middleware registrované po `Run` se nikdy nespustí.

**Další krok:** [Autentizace a autorizace](Auth.md)
