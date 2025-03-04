# JIT (Just-In-Time) vs AOT (Ahead-Of-Time) kompilace v .NET

.NET kompiluje kód dvakrát: jednou z C# do IL (Intermediate Language) při buildu, a podruhé z IL do nativního strojového kódu. Druhý krok může proběhnout buď za běhu aplikace (JIT), nebo předem při publikování (AOT). Volba mezi nimi ovlivňuje dobu startu, výkon za běhu, velikost binárního souboru a kompatibilitu s reflexí.

## 1. Koncept

### JIT (Just-In-Time)

JIT kompiluje metody z IL do nativního kódu **těsně před jejich prvním spuštěním**. To je výchozí chování v .NET. Jakmile je metoda jednou zkompilována, výsledek se cachuje po celý životní cyklus procesu – JIT tedy zaplatíte pouze při prvním volání každé metody.

Hlavní výhoda JITu je, že ví, na jakém hardwaru běží, a může toho využít: může vybrat SIMD instrukce specifické pro konkrétní CPU, může inlinovat metody na základě reálných profilů volání nebo eliminovat větve kódu, které jsou za běhu vždy false. Tuto schopnost AOT ze své podstaty nemá.

### AOT (Ahead-Of-Time)

AOT zkompiluje celý kód do nativního binárního souboru **při publikování** aplikace. Výsledný soubor nevyžaduje .NET runtime na cílovém stroji – vše je zahrnuto. Aplikace se spustí okamžitě, protože žádná kompilace za běhu neprobíhá.

V .NET existují dvě hlavní formy AOT:

| Forma | Popis | Kdy zaveden |
|-------|-------|-------------|
| **ReadyToRun (R2R)** | Předkompiluje část kódu; JIT se použije pro zbytek | .NET Core 3.0 |
| **NativeAOT** | Plná kompilace do nativního exe; žádný JIT, žádný .NET runtime | .NET 7 |

## 2. Příklad

### Publikování jako NativeAOT

```xml
<!-- csproj -->
<PropertyGroup>
  <PublishAot>true</PublishAot>
  <OptimizeSpeed>true</OptimizeSpeed>
</PropertyGroup>
```

```bash
dotnet publish -r linux-x64 -c Release
# Výsledek: jeden nativní binární soubor, ~10–20 MB, spuštění < 10 ms
```

### ReadyToRun – rychlejší start bez omezení reflexe

```bash
dotnet publish -r win-x64 -c Release --self-contained -p:PublishReadyToRun=true
# Předkompiluje většinu kódu; JIT se stále použije pro dynamické části
```

### Porovnání doby startu (orientační, jednoduchá WebAPI)

| Režim | Startup time | Paměť RSS | Velikost souboru |
|-------|-------------|-----------|-----------------|
| JIT (výchozí) | ~300 ms | ~80 MB | ~5 MB + runtime |
| ReadyToRun | ~150 ms | ~75 MB | ~20 MB self-contained |
| NativeAOT | ~10 ms | ~25 MB | ~15 MB |

## 3. Kdy použít

**JIT (výchozí) je správnou volbou pro:**
- Standardní webové API, desktopové aplikace, démon procesy.
- Aplikace, které intenzivně využívají reflexi, `dynamic`, Expression Trees nebo `Activator.CreateInstance`.
- Situace, kde je throughput za běhu důležitější než čas startu.

**ReadyToRun je dobrou volbou, když:**
- Chcete zkrátit startup bez omezení funkcí (reflexe funguje plně).
- Nasazujete do Kubernetes a potřebujete rychlejší scale-out.

**NativeAOT je správnou volbou pro:**
- CLI nástroje, kde každá milisekunda startu záleží.
- Serverless funkce (Azure Functions, AWS Lambda) – platíte za cold start.
- Kontejnery s přísným limitem paměti (IoT, embedded Linux).
- Knihovny distribuované jako nativní `.so`/`.dll` pro jiné jazyky.

## 4. Časté chyby

- ❌ **Použití NativeAOT v aplikaci s reflexí** – reflexe funguje omezeně; dynamicky načítané typy se nemusí kompilovat. Nutné přidávat `[DynamicallyAccessedMembers]` anotace nebo `rd.xml` konfiguraci.
- ❌ **Záměna NativeAOT a SelfContained** – `PublishSelfContained` zabalí .NET runtime do výstupu, ale stále používá JIT. NativeAOT je jiná věc.
- ❌ **Ignorování třímístných výjimek při NativeAOT buildu** – trimmer odstraní kód, na který nenajde statickou referenci. Testujte po každé velké změně.
- ❌ **ReadyToRun pro aplikace s krátkým životem** – R2R má větší binární soubory a plný efekt nastane až po zahřátí; u aplikací, které běží minuty, to nemusí mít smysl.

---

<details>
<summary>Deep dive: jak JIT optimalizuje za běhu a omezení NativeAOT</summary>

### JIT optimalizace, které AOT nedokáže

**Inlining na základě profilu:** JIT sleduje, které metody jsou volány nejčastěji, a rozhoduje o inliningu za běhu. Metoda, která je volána milionkrát, se může inline rozvinout jinak než metoda volaná jednou.

**Devirtualizace:** Pokud JIT vidí, že přes rozhraní je za běhu vždy volána konkrétní implementace, může volání devirtualizovat a inlinovat přímo tuto implementaci – bez virtual dispatch overhead.

**Tiered compilation:** .NET 6+ má dvoustupňovou kompilaci. Při prvním volání se metoda zkompiluje rychle (Tier 0, bez optimalizací). Pokud je metoda volána dostatečně často, JIT ji znovu zkompiluje s plnými optimalizacemi (Tier 1). Startup tedy není brzden pomalou optimalizující kompilací.

```
Tier 0 (rychlá kompilace, žádné optimalizace)
  ↓ pokud je metoda "hot"
Tier 1 (plné optimalizace, PGO data)
```

### NativeAOT omezení

NativeAOT provádí **tree-shaking** (trimming) – odstraní kód, na který nenajde statickou referenci. To znamená:

- `Assembly.GetTypes()` nevrátí všechny typy – některé mohly být odstraněny.
- Serializéry postavené na reflexi (Newtonsoft.Json, XML serializer) nefungují bez konfigurace. Preferujte source-generated serializery (`System.Text.Json` s `[JsonSerializable]`).
- `Activator.CreateInstance(typeof(T))` funguje jen pokud je `T` staticky dosažitelný.

### Jak vybrat pro cloud/kontejnery

```
Mikroservis v Kubernetes s mnoha instancemi?
  → ReadyToRun (dobrý kompromis, žádná omezení)

Azure Function s cold startem?
  → NativeAOT (pokud nepotřebujete reflexi)

Standardní ASP.NET Core API?
  → JIT s Tiered Compilation (výchozí nastavení je správné)

CLI tool distribuovaný jako single file?
  → NativeAOT (malý soubor, okamžitý start)
```

</details>

**Další krok:** [Optimalizace práce s I/O](Optimalizace_IO.md)
