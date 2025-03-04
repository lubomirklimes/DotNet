# .NET CLI a System.CommandLine

`dotnet` CLI je vstupní bod pro všechny operace .NET projektu. `System.CommandLine` umožňuje stavět vlastní CLI nástroje v C# s podporou příkazů, voleb, argumentů a auto-complete.

## 1. Koncept

`dotnet` CLI pokrývá celý životní cyklus projektu: vytvoření, build, test, publish, správu NuGet balíčků. Lze jej rozšířit o **globální nástroje** (`dotnet tool install`).

`System.CommandLine` je Microsoft knihovna pro tvorbu robustních CLI aplikací:
- Hierarchické příkazy (subcommands)
- Silně typované volby a argumenty
- Automatická nápověda (`--help`) a validace
- Tab completion pro bash/zsh/fish

## 2. Příklad

### Nejčastější `dotnet` příkazy

```bash
dotnet new webapi -n MojeApi --use-controllers   # nový projekt
dotnet add package Serilog.AspNetCore             # přidat NuGet
dotnet build -c Release                           # build
dotnet test --logger "trx;LogFileName=out.trx"   # testy
dotnet publish -c Release -o ./publish            # publish
dotnet run --project ./src/MojeApi               # spuštění
dotnet tool install -g dotnet-ef                  # globální nástroj

# EF Core migrace
dotnet ef migrations add InitDb --project ./src/Data
dotnet ef database update
```

### System.CommandLine — jednoduchý CLI nástroj

```csharp
// dotnet add package System.CommandLine
using System.CommandLine;

var inputOpt  = new Option<FileInfo>("--input",  "Vstupní soubor")  { IsRequired = true };
var outputOpt = new Option<FileInfo>("--output", "Výstupní soubor") { IsRequired = true };
var verboseOpt = new Option<bool>("--verbose",   "Podrobný výstup");

var rootCmd = new RootCommand("Konvertor souborů");
rootCmd.AddOption(inputOpt);
rootCmd.AddOption(outputOpt);
rootCmd.AddOption(verboseOpt);

rootCmd.SetHandler(async (input, output, verbose) =>
{
    if (verbose) Console.WriteLine($"Konvertuji {input.Name} → {output.Name}");
    var obsah = await File.ReadAllTextAsync(input.FullName);
    await File.WriteAllTextAsync(output.FullName, obsah.ToUpperInvariant());
}, inputOpt, outputOpt, verboseOpt);

return await rootCmd.InvokeAsync(args);
```

### Subcommands

```csharp
var rootCmd   = new RootCommand("Správce projektu");
var buildCmd  = new Command("build",  "Sestaví projekt");
var deployCmd = new Command("deploy", "Nasadí aplikaci");
var envOpt    = new Option<string>("--env", getDefaultValue: () => "staging");

deployCmd.AddOption(envOpt);
deployCmd.SetHandler(env => Console.WriteLine($"Nasazuji do {env}"), envOpt);

rootCmd.AddCommand(buildCmd);
rootCmd.AddCommand(deployCmd);
return await rootCmd.InvokeAsync(args);
```

## 3. Kdy použít

- **`dotnet` CLI** — vždy pro správu projektů; preferujte skriptování přes CLI před ručními úpravami `.csproj`.
- **`System.CommandLine`** — DevOps nástroje, migrační skripty, vlastní build utility, generátory kódu.
- **Globální nástroje** (`dotnet tool`) — sdílené utility v týmu; publikujte jako NuGet balíček.

## 4. Časté chyby

- ❌ **Ruční editace `.csproj` místo `dotnet add`** — CLI zajistí správné verze a formát; ruční editace snadno způsobí chyby.
- ❌ **`Environment.Exit()` místo návratového kódu** — CLI nástroje musí vracet exit code přes `return`; `Environment.Exit` obejde `finally` bloky.
- ❌ **Synchronní handler v `System.CommandLine`** — `SetHandler` podporuje async; vždy použijte async variantu pro I/O operace.
- ❌ **Bez `--help` testování** — ověřte, že nápověda je srozumitelná a příklady v popisech voleb jsou správné.

**Další krok:** [IoT a embedded s .NET](IoT.md)
