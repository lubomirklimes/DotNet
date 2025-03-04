# Autentizace a autorizace v ASP.NET Core

Autentizace (**kdo jsi?**) a autorizace (**smíš to udělat?**) jsou dvě oddělené fáze zpracování requestu v ASP.NET Core.

## 1. Koncept

| | Autentizace | Autorizace |
|---|---|---|
| Otázka | Kdo je volající? | Má přístup? |
| Middleware | `UseAuthentication()` | `UseAuthorization()` |
| Výsledek | `HttpContext.User` naplněn | Přístup povolen / 403 |
| Hlavní typy | Cookie, JWT Bearer, OAuth | Role, Policy, Claims |

### JWT Bearer – nejběžnější pro API

Token se odesílá v hlavičce: `Authorization: Bearer <token>`. Server ho ověří podpisem – bez DB dotazu.

## 2. Příklad

### JWT Bearer konfigurace

```csharp
// Program.cs
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer           = true,
            ValidIssuer              = builder.Configuration["Jwt:Issuer"],
            ValidateAudience         = true,
            ValidAudience            = builder.Configuration["Jwt:Audience"],
            ValidateLifetime         = true,
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]!))
        };
    });

builder.Services.AddAuthorization();

app.UseAuthentication();
app.UseAuthorization();
```

### Generování JWT tokenu

```csharp
public string VytvorToken(Uzivatel uzivatel, IConfiguration config)
{
    var klic = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(config["Jwt:Key"]!));
    var claims = new[]
    {
        new Claim(ClaimTypes.NameIdentifier, uzivatel.Id.ToString()),
        new Claim(ClaimTypes.Email, uzivatel.Email),
        new Claim(ClaimTypes.Role, uzivatel.Role)
    };

    var token = new JwtSecurityToken(
        issuer:   config["Jwt:Issuer"],
        audience: config["Jwt:Audience"],
        claims:   claims,
        expires:  DateTime.UtcNow.AddHours(1),
        signingCredentials: new SigningCredentials(klic, SecurityAlgorithms.HmacSha256)
    );

    return new JwtSecurityTokenHandler().WriteToken(token);
}
```

### Autorizace – atributy a policy

```csharp
// Role-based
[Authorize(Roles = "Admin,Manager")]
[HttpDelete("/api/produkty/{id}")]
public async Task<IActionResult> Delete(int id) { /* ... */ }

// Policy-based
builder.Services.AddAuthorization(opt =>
{
    opt.AddPolicy("MinVek18", p =>
        p.RequireClaim("vek").RequireAssertion(ctx =>
            int.Parse(ctx.User.FindFirstValue("vek") ?? "0") >= 18));
});

[Authorize(Policy = "MinVek18")]
app.MapGet("/api/obsah-18+", () => "Obsah pro dospělé");

// Povolení anonymního přístupu
[AllowAnonymous]
app.MapGet("/api/verejne", () => "Veřejný endpoint");
```

## 3. Kdy použít

- **JWT** – API bez stavu (stateless), mobilní klienti, SPA, mikroservisy.
- **Cookie autentizace** – tradiční webové aplikace s Razor Pages / MVC.
- **Policy autorizace** – složitá pravidla přístupu (claims, tenant, věk, předplatné).
- **`[AllowAnonymous]`** – explicitně označte veřejné endpointy; výchozí politika může být `RequireAuthenticatedUser`.

## 4. Časté chyby

- ❌ **`UseAuthorization()` bez `UseAuthentication()`** – `HttpContext.User` zůstane anonymní; autorizace vždy selže nebo povolí vše.
- ❌ **Symetrický klíč v appsettings.json** – nikdy necommitujte `Jwt:Key` do repozitáře; použijte User Secrets nebo Azure Key Vault.
- ❌ **Krátká expirace bez refresh tokenu** – uživatelé jsou vyhlášeni každých 15 minut; implementujte refresh token flow.
- ❌ **Role jako string bez enum** – překlep v `"Admim"` místo `"Admin"` způsobí tichý přístup odepřen. Definujte konstanty nebo enum pro role.

**Další krok:** [OpenAPI a Swagger](OpenAPI.md)
