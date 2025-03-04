# .NET IoT

.NET umožňuje psát kód pro embedded a IoT zařízení pomocí balíčků `System.Device.Gpio` a `Iot.Device.Bindings`, které poskytují přístup k GPIO, I²C, SPI a sériovým rozhraním na Raspberry Pi a dalších SBC.

## 1. Koncept

`System.Device.Gpio` je nízkoúrovňová abstrakce nad hardware GPIO piny. `Iot.Device.Bindings` obsahuje přes 200 připravených ovladačů pro senzory, displeje a aktuátory (DHT22, BMP280, SSD1306 OLED, servomotory aj.).

Cílové platformy: Raspberry Pi (Raspberry Pi OS, Linux ARM), BeagleBone, nebo jakékoliv Linux ARM zařízení s GPIO přístupem. Spouštění probíhá jako standardní .NET konzolová aplikace (self-contained publish).

## 2. Příklad

### Blikání LED (GPIO output)

```csharp
// dotnet add package System.Device.Gpio
using System.Device.Gpio;

const int PinLed = 17; // BCM číslo pinu

using var controller = new GpioController();
controller.OpenPin(PinLed, PinMode.Output);

Console.CancelKeyPress += (_, e) =>
{
    e.Cancel = true;
    controller.Write(PinLed, PinValue.Low);
    controller.ClosePin(PinLed);
};

while (true)
{
    controller.Write(PinLed, PinValue.High);
    await Task.Delay(500);
    controller.Write(PinLed, PinValue.Low);
    await Task.Delay(500);
}
```

### Čtení teploty a vlhkosti z DHT22

```csharp
// dotnet add package Iot.Device.Bindings
using Iot.Device.DHTxx;

using var dht = new Dht22(pinNumber: 4); // GPIO 4

while (true)
{
    if (dht.TryReadHumidity(out var vlhkost) && dht.TryReadTemperature(out var teplota))
    {
        Console.WriteLine($"Teplota: {teplota.DegreesCelsius:F1} °C, Vlhkost: {vlhkost.Percent:F1} %");
    }
    else
    {
        Console.WriteLine("Čtení selhalo, zkouším znovu...");
    }
    await Task.Delay(2000);
}
```

### I²C senzor tlaku BMP280

```csharp
using System.Device.I2c;
using Iot.Device.Bmxx80;

var i2cSettings = new I2cConnectionSettings(busId: 1, deviceAddress: Bmp280.DefaultI2cAddress);
using var i2cDevice = I2cDevice.Create(i2cSettings);
using var bmp280 = new Bmp280(i2cDevice);

bmp280.StandbyTime = StandbyTime.Ms250;
await bmp280.SetPowerModeAsync(Bmx280PowerMode.Normal);

var vysledek = await bmp280.ReadAsync();
Console.WriteLine($"Tlak: {vysledek.Pressure?.Hectopascals:F2} hPa");
Console.WriteLine($"Teplota: {vysledek.Temperature?.DegreesCelsius:F1} °C");
```

### Nasazení na Raspberry Pi

```bash
dotnet publish -c Release -r linux-arm64 --self-contained -o ./publish
scp -r ./publish pi@raspberrypi.local:/home/pi/app
ssh pi@raspberrypi.local "chmod +x /home/pi/app/MojeIoTApp && /home/pi/app/MojeIoTApp"
```

## 3. Kdy použít

- **Prototypy a hobby projekty** — rychlé experimentování s senzory bez nutnosti C/C++.
- **Edge computing** — lokální zpracování dat senzorů s odesíláním agregátů do Azure IoT Hub.
- **Průmyslová automatizace** — sběr dat z průmyslových senzorů (Modbus, OPC-UA) přes dostupné NuGet balíčky.

## 4. Časté chyby

- ❌ **Spuštění bez `sudo` nebo bez přístupu ke skupině `gpio`** — na Raspberry Pi OS přidejte uživatele do skupiny `gpio`; `sudo` je nutné jen bez skupinového přístupu.
- ❌ **Nesprávné BCM číslo pinu** — Raspberry Pi má fyzické (BOARD) a BCM čísla; `GpioController` výchozně používá BCM; ověřte pomocí `pinout`.
- ❌ **Chybějící `using` na `GpioController`** — nezavřený controller ponechá piny otevřené; vždy použijte `using` nebo `Dispose()`.
- ❌ **Synchronní `Thread.Sleep` v main loop** — blokuje vlákno a znemožňuje reagovat na CancellationToken; použijte `await Task.Delay(ms, ct)`.

**Další krok:** [AI a ML.NET](ML_NET.md)
