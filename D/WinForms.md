# Windows Forms (WinForms)

WinForms je nejjednodušší způsob, jak vytvořit desktopovou aplikaci s grafickým rozhraním ve Windows.

## Vytvoření projektu

```
dotnet new winforms -n MojeAplikace
```

## Struktura aplikace

WinForms aplikace se skládá z **formulářů** (`Form`), které obsahují **ovládací prvky** (tlačítka, texboxy, labely...).

```csharp
// Program.cs
Application.Run(new MainForm());
```

```csharp
// MainForm.cs
public partial class MainForm : Form
{
    public MainForm()
    {
        InitializeComponent(); // generováno designerem
    }
}
```

## Tlačítko a událost

```csharp
Button btn = new Button
{
    Text = "Klikni na mě",
    Location = new Point(50, 50),
    Size = new Size(120, 30)
};

btn.Click += (sender, e) =>
{
    MessageBox.Show("Ahoj!");
};

this.Controls.Add(btn);
```

## Běžné ovládací prvky

| Prvek | Popis |
|---|---|
| `Button` | Tlačítko |
| `TextBox` | Vstupní pole |
| `Label` | Popisek |
| `ListBox` | Seznam položek |
| `ComboBox` | Rozbalovací seznam |
| `DataGridView` | Tabulka dat |
| `MenuStrip` | Nabídková lišta |

## Práce s formulovými prvky

```csharp
// TextBox
string vstup = textBoxJmeno.Text;

// Label
labelVysledek.Text = $"Ahoj, {vstup}!";

// ListBox
listBoxJmena.Items.Add("Petr");
listBoxJmena.Items.Clear();
string vybrana = listBoxJmena.SelectedItem?.ToString();
```

## Async operace v UI

```csharp
private async void btnNacti_Click(object sender, EventArgs e)
{
    btnNacti.Enabled = false;
    labelStav.Text = "Načítám...";

    var data = await NactiDataAsync();

    listBoxData.Items.AddRange(data.ToArray());
    labelStav.Text = "Hotovo.";
    btnNacti.Enabled = true;
}
```

> UI prvky lze měnit jen z UI vlákna. Při async `await` se vlákno vrátí automaticky.

---

<details>
<summary>Volitelné: Vlastní dialog a data binding</summary>

**Vlastní dialog:**

```csharp
using var dialog = new OsobniFormDialog();
if (dialog.ShowDialog() == DialogResult.OK)
{
    string vysledek = dialog.Vysledek;
}
```

**Data binding na `BindingSource`:**

```csharp
BindingSource bs = new BindingSource();
bs.DataSource = seznam; // List<T>
dataGridView1.DataSource = bs;
```

</details>

---

**Další krok:** [WPF (Windows Presentation Foundation)](WPF.md)
