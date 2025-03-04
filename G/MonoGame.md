# MonoGame – herní framework pro C#

MonoGame je open-source framework pro 2D (a základní 3D) hry v C#. Na rozdíl od Unity neposkytuje vizuální editor – vše píšete v kódu. Za to získáte plnou kontrolu nad herní smyčkou, renderováním a výkonem. MonoGame stojí za hrami jako Celeste, Stardew Valley nebo FEZ.

## 1. Koncept

### Herní smyčka

Každá hra funguje jako nekonečná smyčka: **Update → Draw → Update → Draw...**

- **`Update`** – zpracuje vstup (klávesnice, myš, gamepad), pohne objekty, zkontroluje kolize, aktualizuje stav hry. Volá se fixně (výchozí 60× za sekundu).
- **`Draw`** – vykreslí aktuální stav na obrazovku. Volá se co nejrychleji nebo synchronně s VSync.

`GameTime.ElapsedGameTime` udává, kolik reálného času uběhlo od posledního volání. Pohyb objektů vždy násobte tímto delta časem – hra pak běží stejně rychle bez ohledu na framerate.

### Content Pipeline

MonoGame kompiluje assety (obrázky, zvuky, fonty) do optimalizovaného binárního formátu pomocí **MonoGame Content Builder (MGCB)**. To zajistí efektivní načítání a kompatibilitu na všech platformách.

## 2. Příklad

### Kompletní pohybující se hráč

```csharp
using Microsoft.Xna.Framework;
using Microsoft.Xna.Framework.Graphics;
using Microsoft.Xna.Framework.Input;

public class MojeHra : Game
{
    private GraphicsDeviceManager _graphics;
    private SpriteBatch _spriteBatch;

    private Texture2D _hracTextura;
    private Vector2 _hracPozice;
    private const float Rychlost = 200f; // pixelů za sekundu

    public MojeHra()
    {
        _graphics = new GraphicsDeviceManager(this);
        _graphics.PreferredBackBufferWidth = 800;
        _graphics.PreferredBackBufferHeight = 600;
        Content.RootDirectory = "Content";
        IsMouseVisible = true;
    }

    protected override void LoadContent()
    {
        _spriteBatch = new SpriteBatch(GraphicsDevice);
        // Načte Content/hrac.xnb (zkompilovaný z hrac.png přes MGCB)
        _hracTextura = Content.Load<Texture2D>("hrac");
        _hracPozice = new Vector2(400, 300); // střed obrazovky
    }

    protected override void Update(GameTime gameTime)
    {
        var delta = (float)gameTime.ElapsedGameTime.TotalSeconds;
        var klavesnice = Keyboard.GetState();

        // Pohyb násobíme delta, aby byl framerate-independent
        if (klavesnice.IsKeyDown(Keys.Right)) _hracPozice.X += Rychlost * delta;
        if (klavesnice.IsKeyDown(Keys.Left))  _hracPozice.X -= Rychlost * delta;
        if (klavesnice.IsKeyDown(Keys.Down))  _hracPozice.Y += Rychlost * delta;
        if (klavesnice.IsKeyDown(Keys.Up))    _hracPozice.Y -= Rychlost * delta;

        // Ukončení hry klávesou Escape
        if (klavesnice.IsKeyDown(Keys.Escape)) Exit();

        base.Update(gameTime);
    }

    protected override void Draw(GameTime gameTime)
    {
        GraphicsDevice.Clear(Color.CornflowerBlue);

        // SpriteBatch sdružuje draw volání do jedné GPU operace – efektivní
        _spriteBatch.Begin();
        _spriteBatch.Draw(_hracTextura, _hracPozice, Color.White);
        _spriteBatch.End();

        base.Draw(gameTime);
    }
}
```

### Detekce kolizí (AABB – Axis-Aligned Bounding Box)

```csharp
// Rectangle.Intersects je nejjednodušší a nejrychlejší kolize pro 2D
var hracRect = new Rectangle((int)_hracPozice.X, (int)_hracPozice.Y,
    _hracTextura.Width, _hracTextura.Height);

foreach (var nepritel in _nepratele)
{
    var nepritelRect = new Rectangle((int)nepritel.Pozice.X, (int)nepritel.Pozice.Y,
        nepritel.Sirka, nepritel.Vyska);

    if (hracRect.Intersects(nepritelRect))
        HracZasazen();
}
```

### Zobrazení textu

```csharp
// SpriteFont se kompiluje přes MGCB z fontu
SpriteFont font = Content.Load<SpriteFont>("PixelFont");

// V Draw metodě:
_spriteBatch.Begin();
_spriteBatch.DrawString(font, $"Skóre: {_skore}", new Vector2(10, 10), Color.Yellow);
_spriteBatch.End();
```

## 3. Kdy použít

- Chcete **plnou kontrolu** nad herní smyčkou, renderováním a výkonem – bez abstrakcí Unity nebo Godot.
- Tvoříte **2D hru** s prostými grafikami (pixel art, tile-based mapy, platformery).
- Chcete distribuovat hru na **Windows, macOS, Linux, Android, iOS nebo konzole** z jedné kódbáze.
- Učíte se principy herního vývoje od základů (herní smyčka, sprite batching, collision detection).

## 4. Časté chyby

- ❌ **Pohyb bez delta time** – `pozice.X += 5` pohybuje hráčem rychlostí závislou na FPS; na rychlém počítači létá, na pomalém plazí. Vždy násobte `delta`.
- ❌ **Vytváření nových objektů v `Draw`** – `new SpriteBatch` nebo `new Texture2D` v Draw alokuje každý frame; přetíží GC a způsobí mikrosekundové pauzy. Alokujte v `LoadContent` nebo na začátku scény.
- ❌ **Zapomenuté `base.Update` / `base.Draw`** – MonoGame interně aktualizuje stav (herní komponenty, content pipeline); bez `base` přestane část funkcí fungovat.
- ❌ **Načítání assetů bez Content Pipeline** – načítání surového PNG přes `Texture2D.FromFile` funguje, ale obejde kompilaci a optimalizaci; na mobilech nebo konzolích to může selhat.

---

<details>
<summary>Deep dive: SpriteBatch batching, správa herních stavů a audio</summary>

### Jak SpriteBatch optimalizuje vykreslování

`SpriteBatch.Begin()` / `End()` sdružuje všechny `Draw` volání mezi nimi do jedné GPU draw call (pokud používají stejnou texturu a nastavení). Minimalizujete volání `Begin`/`End` a skupinujte sprity podle textury.

```csharp
// ✅ Efektivní – jedna Begin/End pro všechny sprity
_spriteBatch.Begin(SpriteSortMode.Texture); // Řadí automaticky dle textury
foreach (var entita in _entity)
    _spriteBatch.Draw(entita.Textura, entita.Pozice, Color.White);
_spriteBatch.End();
```

`SpriteSortMode.Texture` interně seřadí sprity a výrazně sníží počet GPU draw calls při mnoha různých texturách.

### Game state management

```csharp
// Jednoduchý stavový stroj pro herní stavy
public enum HerniStav { HlavniMenu, Hra, PauzovaObrazovka, GameOver }

public class MojeHra : Game
{
    private HerniStav _stavHry = HerniStav.HlavniMenu;

    protected override void Update(GameTime gameTime)
    {
        switch (_stavHry)
        {
            case HerniStav.HlavniMenu: AktualizujMenu(gameTime); break;
            case HerniStav.Hra:        AktualizujHru(gameTime); break;
            case HerniStav.GameOver:   AktualizujGameOver(gameTime); break;
        }
        base.Update(gameTime);
    }
}
```

### Audio

```csharp
// Načtení a přehrání zvukového efektu
SoundEffect vybuchy = Content.Load<SoundEffect>("vybuch");
vybuchy.Play(volume: 0.8f, pitch: 0f, pan: 0f);

// Hudba na pozadí
Song hudba = Content.Load<Song>("bgm_level1");
MediaPlayer.Play(hudba);
MediaPlayer.IsRepeating = true;
```

</details>

**Další krok:** [Práce s fyzikou a kolizemi](GamePhysics.md)
