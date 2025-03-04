# Práce s GPS, senzory a notifikacemi (.NET MAUI)

.NET MAUI Essentials nabízí jednotné API pro přístup k hardwarovým funkčnístem bez nápisu platform-specifického kódu.

## GPS a geolokace

```csharp
using Microsoft.Maui.Devices.Sensors;

// Aktuální poloha
var poloha = await Geolocation.GetLastKnownLocationAsync();
if (poloha != null)
    Console.WriteLine($"Lat: {poloha.Latitude}, Lon: {poloha.Longitude}");

// Přesná poloha (vyžaduje opravnění v manifestu)
var request = new GeolocationRequest(GeolocationAccuracy.Medium);
var presna = await Geolocation.GetLocationAsync(request);
```

**Oprávnění (AndroidManifest.xml):**
```xml
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION"/>
```

## Senzory

```csharp
// Akcelerometr
if (Accelerometer.IsSupported)
{
    Accelerometer.ReadingChanged += (s, e) =>
        Console.WriteLine($"X:{e.Reading.Acceleration.X:F2}");
    Accelerometer.Start(SensorSpeed.UI);
}

// Kompas
Compass.ReadingChanged += (s, e) =>
    Console.WriteLine($"Sever: {e.Reading.HeadingMagneticNorth:F0}°");
Compass.Start(SensorSpeed.UI);
```

## Lokální notifikace

```csharp
// NuGet: Plugin.LocalNotification
var notifikace = new NotificationRequest
{
    NotificationId = 1,
    Title = "Upozornění",
    Description = "Nezapomeň na schůzku!",
    Schedule = new NotificationRequestSchedule
    {
        NotifyTime = DateTime.Now.AddMinutes(30)
    }
};

await LocalNotificationCenter.Current.Show(notifikace);
```

## Kamera a galerie

```csharp
// Pořízení fotky
var foto = await MediaPicker.CapturePhotoAsync();
if (foto != null)
{
    using var stream = await foto.OpenReadAsync();
    imageView.Source = ImageSource.FromStream(() => stream);
}

// Výběr z galerie
var vybrana = await MediaPicker.PickPhotoAsync();
```

---

<details>
<summary>Volitelné: Push notifikace a mapa</summary>

**Push notifikace** (FCM / APNs):
- Pro Android: Firebase Cloud Messaging (FCM)
- Pro iOS: Apple Push Notification Service (APNs)
- .NET integrace: `Plugin.Firebase.CloudMessaging`

**Mapa v MAUI:**

```xml
<maps:Map x:Name="MapView" MapType="Street"/>
```

```csharp
var poloha = new Location(50.0755, 14.4378); // Praha
MapView.MoveToRegion(MapSpan.FromCenterAndRadius(poloha, Distance.FromKilometers(5)));
```

</details>

---

**Další krok:** [ASP.NET Core MVC – klasický web](../E/ASP_NET_MVC.md)
