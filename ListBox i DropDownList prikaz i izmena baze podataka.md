# ListBox i DropDownList: prikaz i izmena baze podataka

Oct 28, 2026 · @Vanja

## Cilj časa i ishodi

Na kraju časa učenik ume da napravi stranicu na kojoj bira kontinent u padajućoj listi, vidi njegove države u listi ListBox i dodaje, menja i briše države u bazi. Za svaku izmenu dovoljna je jedna linija C# koda: poziv metode `Insert()`, `Update()` ili `Delete()` kontrole SqlDataSource.

Učenik će umeti da:

- objasni razliku između kontrola DropDownList i ListBox i izabere pravu za zadatak,
- podesi svojstva ListBox-a `Rows` i `SelectionMode` i pročita izabranu stavku (`SelectedIndex`, `SelectedItem`, `SelectedValue`),
- doda kontroli SqlDataSource naredbe INSERT, UPDATE i DELETE sa parametrima koji čitaju vrednosti iz TextBox-a i lista,
- napiše metode za obradu događaja `Click` i `SelectedIndexChanged`,
- proveri unos pre izmene baze i ispiše poruku korisniku.

Preduslovi:

- urađena lekcija **DropDownList i SqlDataSource (Kontinent → Država)**: isti Web Site projekat, ista baza Geografija i isti konekcioni string `GeografijaConnectionString`
- Visual Studio 2010 (Express), projekat ASP.NET Web Site, jezik C#, .NET Framework 4.0
- SQL naredbe INSERT, UPDATE i DELETE

## Teorija: ListBox i DropDownList

Obe kontrole prikazuju listu stavki i potiču od iste osnovne klase `ListControl`, pa se pune i čitaju na isti način. Razlikuju se po izgledu i po broju stavki koje korisnik može da izabere.

|  | DropDownList | ListBox |
| --- | --- | --- |
| Izgled | zatvorena padajuća lista, vidi se samo izabrana stavka | otvorena lista, vidi se više stavki odjednom |
| Koliko stavki se bira | tačno jedna | jedna ili više (svojstvo `SelectionMode`) |
| Izbor na početku | uvek je izabrana prva stavka | ništa nije izabrano, `SelectedIndex = -1` |
| U pregledaču (HTML) | `<select>` | `<select size="4">`, uz `multiple` kada je `SelectionMode = Multiple` |
| Kada se koristi | jedna vrednost, a malo je mesta (kontinent, kategorija, grad) | korisnik treba da vidi spisak i radi sa njim (bira stavku za izmenu, više stavki odjednom) |

### Zajednička svojstva

| Svojstvo | Šta radi |
| --- | --- |
| `Items` | kolekcija stavki; svaka stavka je `ListItem` sa delovima `Text` i `Value` |
| `DataSourceID`, `DataTextField`, `DataValueField` | punjenje iz izvora podataka, isto kao u prethodnoj lekciji |
| `SelectedIndex` | redni broj izabrane stavke, broji se od 0; -1 kad ništa nije izabrano |
| `SelectedItem` | izabrana stavka (`ListItem`); `SelectedItem.Text` je tekst koji korisnik vidi |
| `SelectedValue` | vrednost izabrane stavke kao string, obično ID iz baze |
| `AutoPostBack` | kad je True, izbor stavke odmah šalje stranicu serveru |
| `AppendDataBoundItems` | stavke iz baze dodaje iza stavki upisanih ručno |

Kad korisnik izabere drugu stavku, nastaje događaj `SelectedIndexChanged`. Metoda za obradu tog događaja izvršava se na serveru, pa lista mora da ima `AutoPostBack = True` da bi se metoda izvršila odmah. Dvoklik na listu u pregledu Design pravi tu metodu.

### Svojstva samo za ListBox

| Svojstvo ili metoda | Šta radi |
| --- | --- |
| `Rows` | broj stavki koje se vide bez skrolovanja (podrazumevano 4) |
| `SelectionMode` | `Single` (jedna stavka) ili `Multiple` (više stavki, uz taster Ctrl ili Shift) |
| `GetSelectedIndices()` | vraća niz rednih brojeva svih izabranih stavki |

Kad je izabrano više stavki, `SelectedItem` i `SelectedValue` vraćaju samo prvu izabranu. Sve izabrane stavke čitaju se petljom kroz `Items`:

```csharp
foreach (ListItem stavka in lstDrzave.Items)
{
    if (stavka.Selected)
    {
        // stavka.Text je naziv, stavka.Value je ID
    }
}
```

## Teorija: SqlDataSource menja podatke

SqlDataSource osim upita SELECT može da čuva i naredbe INSERT, UPDATE i DELETE. Svaka naredba ima svoju kolekciju parametara, a izvršava se pozivom odgovarajuće metode iz koda.

| Naredba (svojstvo) | Kolekcija parametara | Šta je pokreće |
| --- | --- | --- |
| `SelectCommand` | `SelectParameters` | lista se puni sama, bez poziva iz koda |
| `InsertCommand` | `InsertParameters` | `sdsDrzave.Insert()` u metodi za dugme Dodaj |
| `UpdateCommand` | `UpdateParameters` | `sdsDrzave.Update()` u metodi za dugme Izmeni |
| `DeleteCommand` | `DeleteParameters` | `sdsDrzave.Delete()` u metodi za dugme Obriši |

Metode `Insert()`, `Update()` i `Delete()` vraćaju broj izmenjenih redova (`int`). Izmenu izvršavaju konekcionim pristupom: SqlDataSource otvori konekciju, izvrši naredbu (`ExecuteNonQuery`) i odmah je zatvori. Svojstvo `DataSourceMode` važi samo za čitanje.

### Parametri iz kontrola

Parametri su isti `ControlParameter` kao u prethodnoj lekciji. Vrednost uzimaju iz svojstva neke kontrole u trenutku kada se pozove metoda.

| Parametar u naredbi | Kontrola | Svojstvo | Šta nosi |
| --- | --- | --- | --- |
| `@Naziv` | `txtNaziv` | `Text` | naziv koji je korisnik upisao |
| `@KontinentID` | `ddlKontinenti` | `SelectedValue` | ID izabranog kontinenta |
| `@DrzavaID` | `lstDrzave` | `SelectedValue` | ID države izabrane u ListBox-u |

### Lista se osvežava sama

Kad `Insert()`, `Update()` ili `Delete()` promeni bar jedan red, SqlDataSource javlja povezanim kontrolama da su se podaci promenili. Lista `lstDrzave` zato ponovo izvršava SELECT pre slanja stranice pregledaču, bez poziva `DataBind()`. Ovo važi samo za kontrole povezane sa istim SqlDataSource-om.

## Forma i kontrole

Radi se u istom Web Site projektu kao prethodna lekcija, na novoj stranici `IzmenaDrzava.aspx`.

1. Desni klik na projekat, pa Add New Item, Web Form, naziv `IzmenaDrzava.aspx`. Zatim desni klik na `IzmenaDrzava.aspx` u Solution Exploreru i Set As Start Page.
2. U pregledu Source nalepi između `<div>` i `</div>` listu kontinenata iz prethodne lekcije, sa uvodnom stavkom i izvorom podataka:

```html
<asp:Label ID="lblKontinent" runat="server" Text="Kontinent:"></asp:Label>
<asp:DropDownList ID="ddlKontinenti" runat="server" AutoPostBack="True"
    AppendDataBoundItems="True" DataSourceID="sdsKontinenti"
    DataTextField="Naziv" DataValueField="KontinentID">
    <asp:ListItem Value="0">-- izaberite kontinent --</asp:ListItem>
</asp:DropDownList>
<asp:SqlDataSource ID="sdsKontinenti" runat="server"
    ConnectionString="<%$ ConnectionStrings:GeografijaConnectionString %>"
    SelectCommand="SELECT [KontinentID], [Naziv] FROM [Kontinenti] ORDER BY [Naziv]">
</asp:SqlDataSource>
<br /><br />
```

3. Pređi u pregled Design. Ispod liste kontinenata prevuci iz Toolbox-a (grupa Standard) kontrole iz tabele, ovim redosledom, i podesi im svojstva u prozoru Properties (F4). Između kontrola pritisni Enter da bi svaka bila u svom redu.

| Kontrola | ID | Svojstva |
| --- | --- | --- |
| ListBox | `lstDrzave` | `Rows = 8`, `Width = 220px`, `AutoPostBack = True` |
| Label | `lblNaziv` | `Text = Naziv države:` |
| TextBox | `txtNaziv` | `Width = 220px` |
| Button | `btnDodaj` | `Text = Dodaj` |
| Button | `btnIzmeni` | `Text = Izmeni` |
| Button | `btnObrisi` | `Text = Obriši` |
| Label | `lblPoruka` | `Text` prazno, `ForeColor = Maroon`, `EnableViewState = False` |

`EnableViewState = False` znači da labela ne pamti tekst između dva slanja stranice. Poruka se zato prikaže jednom i sama nestane pri sledećem kliku.

## Lista država u ListBox-u

ListBox se povezuje sa bazom isto kao DropDownList u prethodnoj lekciji: preko smart taga i čarobnjaka.

1. Klikni na smart tag liste `lstDrzave` i izaberi Choose Data Source, zatim New Data Source, Database. Za ID upiši `sdsDrzave` i klikni OK.
2. Izaberi već sačuvanu konekciju `GeografijaConnectionString` i klikni Next.
3. U koraku Configure the Select Statement izaberi tabelu `Drzave` i kolone `DrzavaID` i `Naziv`.
4. Klikni WHERE i podesi: Column = `KontinentID`, Operator = `=`, Source = Control, Control ID = `ddlKontinenti`. Klikni Add, pa OK.
5. Klikni ORDER BY, izaberi sortiranje po `Naziv`, pa OK i Next. Test Query sa vrednošću 1 mora da pokaže evropske države. Klikni Finish.
6. U prozoru Choose Data Source podesi polje za prikaz na `Naziv` i polje za vrednost na `DrzavaID`, pa OK.
7. Proveri da je na listi uključen `AutoPostBack` (u smart tagu Enable AutoPostBack).

Dugme Advanced u čarobnjaku nudi Generate INSERT, UPDATE, and DELETE statements. Te naredbe su napravljene za GridView i DetailsView, koje same šalju vrednosti. Naša dugmad uzimaju vrednosti iz TextBox-a i lista, pa naredbe pišemo sami u sledećem koraku.

## Naredbe za izmenu

Sve tri naredbe se upisuju u prozoru Command and Parameter Editor. Postupak je isti za sve tri, razlikuju se samo naredba i parametri.

### INSERT, korak po korak

1. U pregledu Design izaberi kontrolu `sdsDrzave` (siva kutija ispod liste) i otvori prozor Properties (F4).
2. Klikni na svojstvo `InsertQuery`, pa na dugme sa tri tačke (...). Otvara se Command and Parameter Editor.
3. U polje INSERT command upiši:

```sql
INSERT INTO Drzave (Naziv, KontinentID) VALUES (@Naziv, @KontinentID)
```

4. Klikni Refresh Parameters. U listi Parameters pojavljuju se `Naziv` i `KontinentID`.
5. Izaberi parametar `Naziv`, pa podesi Parameter source = Control i ControlID = `txtNaziv`.
6. Izaberi parametar `KontinentID`, pa podesi Parameter source = Control i ControlID = `ddlKontinenti`. Klikni OK.

Svojstvo koje parametar čita (`PropertyName`) popunjava se samo: za TextBox je `Text`, a za liste `SelectedValue`.

### UPDATE i DELETE

Na isti način popuni svojstva `UpdateQuery` i `DeleteQuery`:

| Svojstvo | Naredba | Parametri (Parameter source = Control) |
| --- | --- | --- |
| `UpdateQuery` | `UPDATE Drzave SET Naziv = @Naziv WHERE DrzavaID = @DrzavaID` | `Naziv` → `txtNaziv`, `DrzavaID` → `lstDrzave` |
| `DeleteQuery` | `DELETE FROM Drzave WHERE DrzavaID = @DrzavaID` | `DrzavaID` → `lstDrzave` |

Pazi na uslov WHERE. Bez njega UPDATE menja naziv svih država u tabeli, a DELETE briše sve države.

### Šta je editor napisao

U pregledu Source kontrola `sdsDrzave` sada ima četiri naredbe i četiri kolekcije parametara:

```html
<asp:SqlDataSource ID="sdsDrzave" runat="server"
    ConnectionString="<%$ ConnectionStrings:GeografijaConnectionString %>"
    SelectCommand="SELECT [DrzavaID], [Naziv] FROM [Drzave] WHERE ([KontinentID] = @KontinentID) ORDER BY [Naziv]"
    InsertCommand="INSERT INTO Drzave (Naziv, KontinentID) VALUES (@Naziv, @KontinentID)"
    UpdateCommand="UPDATE Drzave SET Naziv = @Naziv WHERE DrzavaID = @DrzavaID"
    DeleteCommand="DELETE FROM Drzave WHERE DrzavaID = @DrzavaID">
    <DeleteParameters>
        <asp:ControlParameter ControlID="lstDrzave" Name="DrzavaID"
            PropertyName="SelectedValue" />
    </DeleteParameters>
    <InsertParameters>
        <asp:ControlParameter ControlID="txtNaziv" Name="Naziv" PropertyName="Text" />
        <asp:ControlParameter ControlID="ddlKontinenti" Name="KontinentID"
            PropertyName="SelectedValue" />
    </InsertParameters>
    <SelectParameters>
        <asp:ControlParameter ControlID="ddlKontinenti" Name="KontinentID"
            PropertyName="SelectedValue" Type="Int32" />
    </SelectParameters>
    <UpdateParameters>
        <asp:ControlParameter ControlID="txtNaziv" Name="Naziv" PropertyName="Text" />
        <asp:ControlParameter ControlID="lstDrzave" Name="DrzavaID"
            PropertyName="SelectedValue" />
    </UpdateParameters>
</asp:SqlDataSource>
```

Isto ime parametra može da postoji u više kolekcija (`KontinentID` je i u `SelectParameters` i u `InsertParameters`). Svaka naredba koristi samo svoju kolekciju.

## Kod: metode za obradu događaja

Dvoklik na dugme u pregledu Design pravi metodu za obradu događaja `Click` i otvara fajl `IzmenaDrzava.aspx.cs`. Dvoklik na ListBox pravi metodu za događaj `SelectedIndexChanged`. Napravi tako sve četiri metode, pa ih popuni:

```csharp
public partial class IzmenaDrzava : System.Web.UI.Page
{
    protected void Page_Load(object sender, EventArgs e)
    {
    }

    // izbor države u listi: naziv se prepisuje u TextBox
    protected void lstDrzave_SelectedIndexChanged(object sender, EventArgs e)
    {
        txtNaziv.Text = lstDrzave.SelectedItem.Text;
    }

    protected void btnDodaj_Click(object sender, EventArgs e)
    {
        if (ddlKontinenti.SelectedValue == "0")
        {
            lblPoruka.Text = "Prvo izaberite kontinent.";
            return;
        }
        if (txtNaziv.Text.Trim() == "")
        {
            lblPoruka.Text = "Upišite naziv države.";
            return;
        }
        sdsDrzave.Insert();
        lblPoruka.Text = "Država " + txtNaziv.Text + " je dodata.";
        txtNaziv.Text = "";
    }

    protected void btnIzmeni_Click(object sender, EventArgs e)
    {
        if (lstDrzave.SelectedIndex == -1)
        {
            lblPoruka.Text = "Izaberite državu u listi.";
            return;
        }
        if (txtNaziv.Text.Trim() == "")
        {
            lblPoruka.Text = "Upišite novi naziv države.";
            return;
        }
        sdsDrzave.Update();
        lblPoruka.Text = "Naziv države je izmenjen.";
        txtNaziv.Text = "";
    }

    protected void btnObrisi_Click(object sender, EventArgs e)
    {
        if (lstDrzave.SelectedIndex == -1)
        {
            lblPoruka.Text = "Izaberite državu u listi.";
            return;
        }
        string naziv = lstDrzave.SelectedItem.Text;
        sdsDrzave.Delete();
        lblPoruka.Text = "Država " + naziv + " je obrisana.";
        txtNaziv.Text = "";
    }
}
```

### Zašto provera pre izmene

- Prazan TextBox šalje u bazu NULL, a kolona `Naziv` ne sme biti prazna (NOT NULL). Bez provere stranica prijavljuje grešku.
- Uvodna stavka ima vrednost 0, a kontinent sa ID-om 0 ne postoji. Strani ključ odbija takav unos.
- U ListBox-u na početku ništa nije izabrano (`SelectedIndex = -1`). Tada nema koje države da se menja ili briše.

Naziv za poruku o brisanju čuvamo u promenljivoj `naziv` pre poziva `Delete()`, jer se posle brisanja lista puni iznova i izabrane stavke više nema.

### Potvrda brisanja

Izaberi dugme `btnObrisi` i u prozoru Properties upiši u svojstvo `OnClientClick`:

```javascript
return confirm('Da li sigurno želite da obrišete izabranu državu?');
```

Ovo je JavaScript koji se izvršava u pregledaču, pre slanja stranice. Ako korisnik klikne Cancel, `confirm` vraća `false` i stranica se ne šalje serveru, pa se `btnObrisi_Click` ne izvršava.

### Šta se promenilo u ASPX-u

```html
<asp:ListBox ID="lstDrzave" runat="server" AutoPostBack="True"
    DataSourceID="sdsDrzave" DataTextField="Naziv" DataValueField="DrzavaID"
    Rows="8" Width="220px"
    onselectedindexchanged="lstDrzave_SelectedIndexChanged"></asp:ListBox>
...
<asp:Button ID="btnDodaj" runat="server" Text="Dodaj" onclick="btnDodaj_Click" />
<asp:Button ID="btnIzmeni" runat="server" Text="Izmeni" onclick="btnIzmeni_Click" />
<asp:Button ID="btnObrisi" runat="server" Text="Obriši" onclick="btnObrisi_Click"
    OnClientClick="return confirm('Da li sigurno želite da obrišete izabranu državu?');" />
```

Atributi `onclick` i `onselectedindexchanged` povezuju kontrolu sa metodom u kodu. Ako se metoda preimenuje ili obriše samo u kodu, a atribut ostane, stranica se neće pokrenuti.

## Kako to radi i provera rada

Cela stranica stoji na dve veze: dugme poziva jednu metodu SqlDataSource-a, a parametri te naredbe čitaju vrednosti iz kontrola u trenutku poziva.

&#91;embedded content: tok jednog klika na Dodaj · 1 provera, 1 linija koda\]

Izmeni i Obriši rade na isti način, samo pozivaju `Update()` i `Delete()`, a `@DrzavaID` se čita iz `lstDrzave.SelectedValue`.

### Provera rada

1. Pokreni stranicu sa F5. Lista kontinenata je puna, a ListBox je prazan.
2. Izaberi Evropu. U ListBox-u se vidi pet evropskih država.
3. Upiši `Grčka` i klikni Dodaj. Pojavljuje se poruka, a Grčka stoji u listi po abecedi.
4. Klikni na Nemačku u listi. Naziv se prepisuje u TextBox. Promeni ga u `Savezna Republika Nemačka` i klikni Izmeni.
5. Izaberi Grčku i klikni Obriši. Na pitanje prvo klikni Cancel: ništa se ne briše. Ponovi i klikni OK: Grčka nestaje iz liste.
6. Isprobaj greške: Dodaj sa praznim TextBox-om, Dodaj sa izabranom uvodnom stavkom kontinenta, Izmeni bez izabrane države. Umesto greške treba da se pojavi poruka.
7. U SQL Server Management Studiju pokreni `SELECT * FROM Drzave` i proveri da su izmene zaista u bazi.
8. Za radoznale: u pregledaču izaberi View Source. ListBox je element `<select size="8">`, a potvrda brisanja je atribut `onclick` sa funkcijom `confirm`.

## Zadaci za vežbu

Zadaci se rade na stranici `IzmenaDrzava.aspx`. Pre napredna dva zadatka sačuvaj kopiju stranice koja radi.

1. **Osnovni.** Kad korisnik promeni kontinent, obriši tekst iz `txtNaziv`. Uputstvo: dvoklik na `ddlKontinenti` pravi metodu za obradu događaja `SelectedIndexChanged`.
2. **Osnovni.** Podesi na `lstDrzave` svojstvo `SelectionMode = Multiple`. Izaberi dve države uz taster Ctrl i klikni Obriši. Koliko država je obrisano i zašto? Posle provere vrati `SelectionMode = Single`.
3. **Srednji.** Ispod ListBox-a dodaj labelu `lblBroj` koja prikazuje koliko država ima izabrani kontinent. Uputstvo: u prozoru Properties liste, na kartici Events (ikonica munje), dvoklik na događaj `DataBound`; broj stavki je `lstDrzave.Items.Count`.
4. **Srednji.** Ispod forme dodaj GridView koji prikazuje sve države sa nazivom kontinenta (novi SqlDataSource, upit sa JOIN). Posle dodavanja države GridView ostaje stari. Objasni zašto i popravi to pozivom `DataBind()` na GridView-u.
5. **Napredni.** Dodaj drugu padajuću listu `ddlNoviKontinent` i dugme Premesti, koje izabranu državu premesti na drugi kontinent. Uputstvo: SqlDataSource ima samo jednu naredbu UPDATE, pa ti treba nova kontrola `sdsPremesti` sa naredbom `UPDATE Drzave SET KontinentID = @NoviKontinentID WHERE DrzavaID = @DrzavaID`. Lista `lstDrzave` se tada ne osvežava sama.
6. **Napredni.** Prepravi dugme Obriši tako da briše sve izabrane države (`SelectionMode = Multiple`). Uputstvo: u `DeleteParameters` promeni parametar `DrzavaID` na Parameter source = None; u petlji kroz `Items` za svaku izabranu stavku postavi `sdsDrzave.DeleteParameters["DrzavaID"].DefaultValue = stavka.Value;` i pozovi `sdsDrzave.Delete()`.

## Pitanja za proveru razumevanja

- Po čemu se ListBox razlikuje od DropDownList? Navedi primer za svaku kontrolu.
- Šta vraća `SelectedIndex` kad u ListBox-u ništa nije izabrano? Može li se to desiti kod DropDownList?
- U kojim svojstvima SqlDataSource čuva naredbe za izmenu i kojim metodama se one izvršavaju?
- Iz koje kontrole i kog svojstva parametar `@DrzavaID` uzima vrednost? Zašto ID, a ne naziv države?
- Zašto se lista država osveži posle klika na Dodaj iako nismo pozvali `DataBind()`?
- Šta bi se desilo da naredba DELETE nema uslov WHERE?

## Česte greške i kako ih rešiti

| Šta se dešava | Najčešći uzrok | Rešenje |
| --- | --- | --- |
| Greška Cannot insert the value NULL into column 'Naziv' | prazan TextBox, a provera ne postoji | pre `Insert()` proveri `txtNaziv.Text.Trim() == ""` |
| Greška The INSERT statement conflicted with the FOREIGN KEY constraint | izabrana je uvodna stavka sa vrednošću 0 | pre `Insert()` proveri `ddlKontinenti.SelectedValue == "0"` |
| Greška Inserting is not supported by data source 'sdsDrzave' unless InsertCommand is specified | svojstvo `InsertQuery` nije popunjeno (isto za Updating i Deleting) | upiši naredbu u `InsertQuery` kontrole `sdsDrzave` |
| Greška Must declare the scalar variable "@Naziv" | parametar ne postoji u kolekciji ili mu je ime drugačije | u editoru klikni Refresh Parameters; ime posle `@` mora biti isto kao `Name` parametra |
| Izmeni i Obriši ne rade ništa, a nema greške | parametar `DrzavaID` čita pogrešnu kontrolu | proveri `ControlID="lstDrzave"` i `PropertyName="SelectedValue"` |
| Izbor države ne prepisuje naziv u TextBox | `AutoPostBack` nije uključen na `lstDrzave` | podesi `AutoPostBack = True`; u ASPX-u mora postojati `onselectedindexchanged` |
| Izmena promeni naziv svim državama | naredba UPDATE nema uslov WHERE | dodaj `WHERE DrzavaID = @DrzavaID` |
| Stara poruka ostaje i posle sledećeg klika | labela pamti tekst u ViewState-u | na `lblPoruka` podesi `EnableViewState = False` |
| Brisanje ne pita za potvrdu | `OnClientClick` nije podešen ili mu nedostaje `return` | `OnClientClick="return confirm('...');"` |
