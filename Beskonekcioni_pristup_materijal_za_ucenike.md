# Beskonekcioni pristup bazi podataka u ASP.NET aplikaciji

**Materijal za učenike · Web programiranje · Visual Studio 2010, C#, MS SQL Server**

U ovoj lekciji učimo kako web aplikacija može da preuzme podatke iz baze, radi sa njima u memoriji i vrati izmene u bazu. Prvo sve radimo klikovima (Primer 1), a zatim isto to radimo kodom (Primer 2).

## Sadržaj

1. [Šta ćemo naučiti](#1-šta-ćemo-naučiti)
2. [Priprema: baza `skola21oktobar`](#2-priprema-baza-skola21oktobar)
3. [Konekcioni i beskonekcioni pristup](#3-konekcioni-i-beskonekcioni-pristup)
4. [Primer 1: GridView i SqlDataSource](#4-primer-1-gridview-i-sqldatasource)
5. [ADO.NET objekti](#5-adonet-objekti)
6. [Primer 2: isti posao u kodu](#6-primer-2-isti-posao-u-kodu)
7. [Česte greške](#7-česte-greške)
8. [Zadaci za vežbu](#8-zadaci-za-vežbu)
9. [Rezime i pitanja za ponavljanje](#9-rezime-i-pitanja-za-ponavljanje)

---

## 1. Šta ćemo naučiti

1. Objasniti razliku između konekcionog i beskonekcionog pristupa bazi.
2. Povezati GridView sa bazom pomoću kontrole SqlDataSource.
3. Uključiti automatski Insert, Update i Delete i pročitati kod koji kontrola doda.
4. Napraviti konekcioni string u Settings-u i popuniti DataTable i DataSet metodom Fill.
5. Izmeniti podatke u memoriji i snimiti ih u bazu metodom Update.

---

## 2. Priprema: baza `skola21oktobar`

Svi primeri koriste bazu **skola21oktobar** sa dve tabele. Bazu pravimo SQL skriptom.

### 2.1 Kako se pokreće skripta

1. Otvori **SQL Server Management Studio** i prijavi se sa **Windows Authentication**. Zapamti **Server name**, trebaće ti kasnije u konekcionom stringu.
2. Klikni **New Query**.
3. Nalepi celu skriptu iz odeljka 2.2.
4. Klikni **Execute** (ili pritisni **F5**).
5. Na dnu treba da se vide dve tabele sa podacima: Odeljenja (4 reda) i Ucenici (12 redova).

Ako dobiješ poruku da nemaš dozvolu za pravljenje baze (*CREATE DATABASE permission denied*), javi se nastavniku.

### 2.2 SQL skripta

```sql
-- ============================================================
-- Baza skola21oktobar: tabele Odeljenja i Ucenici
-- Pokretanje: SQL Server Management Studio > New Query > Execute (F5)
-- Skripta se sme pokrenuti više puta: tabele se brišu i prave iznova.
-- ============================================================

-- 1. Pravljenje baze (ako ne postoji)
IF DB_ID(N'skola21oktobar') IS NULL
    CREATE DATABASE skola21oktobar;
GO

USE skola21oktobar;
GO

-- 2. Brisanje starih tabela (prvo Ucenici, jer ima strani ključ ka Odeljenja)
IF OBJECT_ID(N'dbo.Ucenici', N'U') IS NOT NULL
    DROP TABLE dbo.Ucenici;
IF OBJECT_ID(N'dbo.Odeljenja', N'U') IS NOT NULL
    DROP TABLE dbo.Odeljenja;
GO

-- 3. Tabela Odeljenja
CREATE TABLE dbo.Odeljenja
(
    Id    INT IDENTITY(1,1) NOT NULL,
    Naziv NVARCHAR(20)      NOT NULL,
    CONSTRAINT PK_Odeljenja PRIMARY KEY (Id)
);
GO

-- 4. Tabela Ucenici
CREATE TABLE dbo.Ucenici
(
    Id          INT IDENTITY(1,1) NOT NULL,
    Ime         NVARCHAR(50)      NOT NULL,
    Prezime     NVARCHAR(50)      NOT NULL,
    OdeljenjeId INT               NOT NULL,
    CONSTRAINT PK_Ucenici PRIMARY KEY (Id),
    CONSTRAINT FK_Ucenici_Odeljenja FOREIGN KEY (OdeljenjeId)
        REFERENCES dbo.Odeljenja (Id)
);
GO

-- 5. Početni podaci (N ispred teksta čuva slova č, ć, š, ž, đ)
INSERT INTO dbo.Odeljenja (Naziv) VALUES (N'III-1');
INSERT INTO dbo.Odeljenja (Naziv) VALUES (N'III-2');
INSERT INTO dbo.Odeljenja (Naziv) VALUES (N'III-3');
INSERT INTO dbo.Odeljenja (Naziv) VALUES (N'III-4');
GO

INSERT INTO dbo.Ucenici (Ime, Prezime, OdeljenjeId) VALUES (N'Marko',    N'Jovanović', 1);
INSERT INTO dbo.Ucenici (Ime, Prezime, OdeljenjeId) VALUES (N'Jelena',   N'Petrović',  2);
INSERT INTO dbo.Ucenici (Ime, Prezime, OdeljenjeId) VALUES (N'Nikola',   N'Ilić',      1);
INSERT INTO dbo.Ucenici (Ime, Prezime, OdeljenjeId) VALUES (N'Ana',      N'Kovač',     3);
INSERT INTO dbo.Ucenici (Ime, Prezime, OdeljenjeId) VALUES (N'Milica',   N'Nikolić',   2);
INSERT INTO dbo.Ucenici (Ime, Prezime, OdeljenjeId) VALUES (N'Stefan',   N'Đorđević',  3);
INSERT INTO dbo.Ucenici (Ime, Prezime, OdeljenjeId) VALUES (N'Teodora',  N'Marković',  1);
INSERT INTO dbo.Ucenici (Ime, Prezime, OdeljenjeId) VALUES (N'Luka',     N'Popović',   2);
INSERT INTO dbo.Ucenici (Ime, Prezime, OdeljenjeId) VALUES (N'Katarina', N'Milošević', 3);
INSERT INTO dbo.Ucenici (Ime, Prezime, OdeljenjeId) VALUES (N'Petar',    N'Stanković', 1);
INSERT INTO dbo.Ucenici (Ime, Prezime, OdeljenjeId) VALUES (N'Vuk',      N'Pavlović',  2);
INSERT INTO dbo.Ucenici (Ime, Prezime, OdeljenjeId) VALUES (N'Sofija',   N'Bogdanović', 3);
GO

-- 6. Provera
SELECT * FROM dbo.Odeljenja;
SELECT * FROM dbo.Ucenici;
GO
```

### 2.3 Struktura tabela

**Odeljenja**

| Kolona | Tip | Opis |
|---|---|---|
| `Id` | `INT IDENTITY` | primarni ključ, broji se automatski |
| `Naziv` | `NVARCHAR(20)` | naziv odeljenja, na primer III-1 |

**Ucenici**

| Kolona | Tip | Opis |
|---|---|---|
| `Id` | `INT IDENTITY` | primarni ključ, broji se automatski |
| `Ime` | `NVARCHAR(50)` | ime učenika |
| `Prezime` | `NVARCHAR(50)` | prezime učenika |
| `OdeljenjeId` | `INT` | strani ključ: mora postojati u tabeli Odeljenja |

### 2.4 Korisno je znati

- Kolona **Id** je *identity*: baza je popunjava sama, zato je ne navodimo u `INSERT` naredbi.
- Tabela **Ucenici** ima strani ključ ka tabeli **Odeljenja**. Ne možeš dodati učenika u odeljenje koje ne postoji, niti obrisati odeljenje koje ima učenike.
- Odeljenje **III-4** namerno nema učenika, pa na njemu možeš bezbedno da probaš brisanje.
- Slova č, ć, š, ž i đ čuvaju se zato što su kolone tipa `NVARCHAR`, a tekst u skripti ima slovo `N` ispred navodnika.
- Skriptu možeš pokrenuti ponovo kad god želiš da vratiš početne podatke: stare tabele se brišu i prave iznova.

---

## 3. Konekcioni i beskonekcioni pristup

| Osobina | Konekcioni (DataReader) | Beskonekcioni (DataSet) |
|---|---|---|
| Veza sa bazom | otvorena dok se čita | otvorena samo za Fill i Update |
| Smer čitanja | samo unapred | bilo kojim redosledom |
| Izmena podataka | nije moguća | moguća, pa Update u bazu |
| Memorija | malo zauzeća | cela tabela u memoriji |
| Kada se koristi | brzo čitanje velikih izveštaja | izmene, više tabela, vezivanje kontrola |

**Osnovna ideja beskonekcionog pristupa:**

1. Preuzmi podatke iz baze.
2. Prekini vezu sa bazom.
3. Radi sa podacima u memoriji.
4. Ponovo se poveži i vrati izmene u bazu.

Veza sa bazom traje samo dok se podaci preuzimaju ili vraćaju.

---

## 4. Primer 1: GridView i SqlDataSource

U ovom primeru **ne pišemo nijednu liniju koda**. Sve podešavamo klikovima, a Visual Studio sam generiše kod.

### 4.1 Povezivanje GridView-a sa bazom

1. Napravi novu web formu. Iz **Toolbox-a** (grupa **Data**) prevuci **GridView** na formu.
2. Klikni na malu strelicu (smart tag) u gornjem desnom uglu kontrole, pa izaberi **Choose Data Source › New data source › Database › OK**.
3. U prozoru **New Connection** izaberi **Microsoft SQL Server (SqlClient)**, upiši **Server name**, izaberi **Use Windows Authentication** i bazu **skola21oktobar**. Klikni **Test Connection**, pa **OK** i **Next**.
4. Sačuvaj connection string u Web.config pod nazivom **SkolaConnectionString**. Čarobnjak predlaže drugi naziv, zato ga promeni. Klikni **Next**.
5. U koraku **Configure the Select Statement** izaberi tabelu **Ucenici** i zvezdicu **(\*)** da se izaberu sve kolone. Klikni **Next**, pa **Test Query** i **Finish**.
6. Pokreni stranicu (**Ctrl+F5**) i proveri da li se podaci prikazuju.

### 4.2 Kod koji je nastao

Pređi na **Source** prikaz stranice (traka na dnu: Design / Split / **Source**) i pogledaj šta je čarobnjak napisao umesto tebe. Redosled atributa može malo da se razlikuje od prikazanog.

**Default.aspx**

```html
<asp:GridView ID="GridView1" runat="server" AutoGenerateColumns="False"
    DataKeyNames="Id" DataSourceID="SqlDataSource1">
    <Columns>
        <asp:BoundField DataField="Id" HeaderText="Id" ReadOnly="True"
            InsertVisible="False" SortExpression="Id" />
        <asp:BoundField DataField="Ime" HeaderText="Ime" SortExpression="Ime" />
        <asp:BoundField DataField="Prezime" HeaderText="Prezime" SortExpression="Prezime" />
        <asp:BoundField DataField="OdeljenjeId" HeaderText="OdeljenjeId"
            SortExpression="OdeljenjeId" />
    </Columns>
</asp:GridView>
<asp:SqlDataSource ID="SqlDataSource1" runat="server"
    ConnectionString="<%$ ConnectionStrings:SkolaConnectionString %>"
    SelectCommand="SELECT * FROM [Ucenici]">
</asp:SqlDataSource>
```

- `DataSourceID` povezuje GridView sa kontrolom `SqlDataSource1`.
- `SqlDataSource1` pamti connection string (`ConnectionString`) i SQL naredbu (`SelectCommand`).

**Web.config** (odeljak `connectionStrings`)

```xml
<connectionStrings>
  <add name="SkolaConnectionString"
       connectionString="Data Source=SERVER;Initial Catalog=skola21oktobar;Integrated Security=True"
       providerName="System.Data.SqlClient" />
</connectionStrings>
```

- `SkolaConnectionString` je ime pod kojim se string pronalazi.
- `SqlDataSource` ga čita iz Web.config-a preko izraza `<%$ ConnectionStrings:... %>`.
- Server ili bazu menjaš samo na ovom jednom mestu.
- `Integrated Security=True` znači da se prijavljuješ Windows nalogom.
- U tvom fajlu `SERVER` je naziv tvog SQL Server-a.

### 4.3 Automatski Insert, Update i Delete

1. Klikni smart tag kontrole **SqlDataSource1** › **Configure Data Source** › **Next**.
2. Klikni **Advanced** i označi **Generate INSERT, UPDATE, and DELETE statements**. Opciju *Use optimistic concurrency* ostavi isključenu. Klikni **OK**, **Next** i **Finish**.
3. Klikni smart tag kontrole **GridView1** i označi **Enable Editing** i **Enable Deleting**.
4. Pokreni stranicu i isprobaj dugmad **Edit**, **Update**, **Cancel** i **Delete**.

> Opcija *Advanced* je neaktivna ako tabela nema primarni ključ. Tabela Ucenici ga ima (kolona `Id`).

Vrati se u **Source** prikaz i uporedi sa prethodnim stanjem. Kontrola je sama dodala:

**1. Dugmad u GridView-u**

```html
<asp:CommandField ShowEditButton="True" ShowDeleteButton="True" />
```

**2. SQL naredbe u SqlDataSource-u**

| Atribut | SQL naredba |
|---|---|
| `SelectCommand` | `SELECT * FROM [Ucenici]` |
| `InsertCommand` | `INSERT INTO [Ucenici] ([Ime], [Prezime], [OdeljenjeId]) VALUES (@Ime, @Prezime, @OdeljenjeId)` |
| `UpdateCommand` | `UPDATE [Ucenici] SET [Ime] = @Ime, [Prezime] = @Prezime, [OdeljenjeId] = @OdeljenjeId WHERE [Id] = @Id` |
| `DeleteCommand` | `DELETE FROM [Ucenici] WHERE [Id] = @Id` |

Reči sa znakom `@` su **parametri**. Njihove vrednosti dolaze iz GridView-a. `INSERT` ne sadrži kolonu `Id` jer je to identity kolona, a u `WHERE` delu naredbi `UPDATE` i `DELETE` koristi se primarni ključ.

**3. Parametri** (ovde je prikazan samo `UpdateParameters`; `InsertParameters` izgledaju slično, a `DeleteParameters` ima samo parametar `Id`)

```html
<UpdateParameters>
    <asp:Parameter Name="Ime" Type="String" />
    <asp:Parameter Name="Prezime" Type="String" />
    <asp:Parameter Name="OdeljenjeId" Type="Int32" />
    <asp:Parameter Name="Id" Type="Int32" />
</UpdateParameters>
```

Kada klikneš **Update**, GridView prosleđuje vrednosti iz polja parametrima, a `Id` uzima iz `DataKeyNames`. SqlDataSource ih ubacuje u `UpdateCommand`.

> **Insert:** GridView ne dodaje nove redove. Za unos novog reda koristi se kontrola **DetailsView** sa istim `SqlDataSource1` (u njenom smart tagu označi *Enable Inserting*).
>
> **Probaj brisanje:** učenika možeš da obrišeš bez problema. Odeljenje koje ima učenike ne možeš (strani ključ), a odeljenje III-4 možeš.

### 4.4 Šta SqlDataSource radi iza kulisa

| Objekat | Šta radi |
|---|---|
| `SqlConnection` | otvara vezu sa bazom na osnovu connection string-a |
| `SqlCommand` | izvršava Select, Insert, Update i Delete naredbe |
| `SqlDataAdapter` | učitava podatke u DataSet (podrazumevani `DataSourceMode`) |

U Primeru 2 isti posao radimo sami, u kodu.

---

## 5. ADO.NET objekti

| Provajder podataka (`using System.Data.SqlClient;`) | Objekti u memoriji (`using System.Data;`) |
|---|---|
| `SqlConnection` – veza sa bazom | `DataSet` – skup tabela |
| `SqlCommand` – SQL naredba | `DataTable` – jedna tabela |
| `SqlDataReader` – čitanje unapred | `DataRow` – jedan red |
| **`SqlDataAdapter` – most ka memoriji** | `DataColumn` – jedna kolona |

`SqlDataAdapter` spaja oba dela: **Fill** puni objekte u memoriji, a **Update** vraća izmene u bazu.

### 5.1 SqlDataAdapter

Adapter ima četiri naredbe: `SelectCommand` (čita podatke), `InsertCommand` (dodaje red), `UpdateCommand` (menja red) i `DeleteCommand` (briše red). Za `Fill` je potrebna samo `SelectCommand`, a preostale tri za `Update`. Možemo ih zadati ručno ili prepustiti `SqlCommandBuilder`-u da ih napravi na osnovu `SELECT` naredbe.

| Metoda | Šta radi |
|---|---|
| `Fill(dt)` | podaci iz baze idu u DataTable ili DataSet |
| `Update(dt)` | izmene iz DataTable-a ili DataSet-a idu u bazu |

### 5.2 DataTable

`DataTable` ne zna ništa o bazi: to je samo skup redova i kolona u memoriji. Posle `Fill` iz tabele Ucenici izgleda ovako:

| Id | Ime | Prezime | OdeljenjeId |
|---|---|---|---|
| 1 | Marko | Jovanović | 1 |
| 2 | Jelena | Petrović | 2 |
| 3 | Nikola | Ilić | 1 |
| 4 | Ana | Kovač | 3 |
| … | … | … | … |

| Izraz | Značenje |
|---|---|
| `dt.Columns` | kolone: Id, Ime, Prezime, OdeljenjeId |
| `dt.Rows` | redovi, svaki red je jedan `DataRow` |
| `dt.Rows[1]["Ime"]` | vrednost ćelije: Jelena (brojanje redova počinje od nule) |

### 5.3 Šta radi metoda Fill

1. Otvara vezu sa bazom (ako je zatvorena).
2. Izvršava `SELECT` naredbu adaptera.
3. Pravi kolone i puni `DataTable` redovima.
4. Zatvara vezu (ako ju je sam otvorio).

`Fill` sam upravlja vezom, pa **ne pozivamo** `Open()` i `Close()`.

---

## 6. Primer 2: isti posao u kodu

Isti rezultat kao u Primeru 1, ali sada sve radimo sami, u kodu.

> Napravi **novu web formu** sa praznim `GridView1`. Ako bi GridView ostao povezan sa `SqlDataSource1` iz Primera 1, javila bi se greška *Both DataSource and DataSourceID are defined* (vidi odeljak 7).

### 6.1 Konekcioni string u Settings-u

1. Izaberi **Project › Properties** (ili desni klik na projekat u Solution Explorer-u).
2. Otvori karticu **Settings**.
3. Upiši **Name**, izaberi **Type** i upiši **Value**.
4. Sačuvaj promene (**Ctrl+S**).

| Polje | Vrednost |
|---|---|
| **Name** | `SkolaConnectionString` |
| **Type** | `(Connection string)` |
| **Scope** | `Application` |
| **Value** | `Data Source=SERVER;Initial Catalog=skola21oktobar;Integrated Security=True` |

U polju **Value** zameni `SERVER` nazivom svog SQL Server-a (isti naziv koji upisuješ pri prijavi u Management Studio).

| Deo stringa | Značenje |
|---|---|
| `Data Source` | server (računar ili instanca), na primer `.`, `(local)` ili `RACUNAR\SQLEXPRESS` |
| `Initial Catalog` | naziv baze: `skola21oktobar` |
| `Integrated Security=True` | prijava Windows nalogom (Windows autentifikacija) |

Kada sačuvaš promene, Visual Studio sam pravi klasu `Properties.Settings` (fajl `Settings.Designer.cs`), preko koje se string čita u kodu.

### 6.2 Čitanje stringa iz koda

```csharp
using System.Data;
using System.Data.SqlClient;

SqlConnection con = new SqlConnection();
con.ConnectionString =
    Properties.Settings.Default.SkolaConnectionString;
```

| Deo koda | Značenje |
|---|---|
| `Settings.Default` | podešavanja iz kartice Settings |
| `SkolaConnectionString` | Name koji smo upisali |
| `ConnectionString` | svojstvo veze koje prima vrednost |

Server ili bazu menjamo samo u Settings-u, a kod ostaje isti. Ako se tvoj projekat zove drugačije i kompajler ne pronalazi `Properties`, navedi ceo naziv: `NazivProjekta.Properties.Settings.Default`.

### 6.3 Fill i prikaz u GridView-u

Ceo kod za događaj `Page_Load`:

```csharp
using System.Data;
using System.Data.SqlClient;

protected void Page_Load(object sender, EventArgs e)
{
    if (!IsPostBack)
    {
        SqlConnection con = new SqlConnection();
        con.ConnectionString =
            Properties.Settings.Default.SkolaConnectionString;

        SqlDataAdapter da = new SqlDataAdapter("SELECT * FROM Ucenici", con);

        DataTable dt = new DataTable();
        da.Fill(dt);

        GridView1.DataSource = dt;
        GridView1.DataBind();
    }
}
```

- Adapter dobija `SELECT` naredbu i `SqlConnection` (`con`).
- `Fill` puni `DataTable` redovima iz baze.
- GridView povezujemo kodom (`DataSource` i `DataBind`), a ne svojstvom `DataSourceID`.

(`using` naredbe idu na sam vrh fajla, a ne unutar metode.)

### 6.4 Čitanje podataka iz DataTable-a

```csharp
foreach (DataRow red in dt.Rows)
{
    string ime = red["Ime"].ToString();
    int id = (int)red["Id"];
    Label1.Text += id + ". " + ime + "<br />";
}

int broj = dt.Rows.Count;
```

| Izraz | Značenje |
|---|---|
| `red["Ime"]` | vrednost po nazivu kolone |
| `red[0]` | vrednost po rednom broju kolone |
| `dt.Rows.Count` | broj redova u tabeli |

Vrednosti u ćelijama su tipa `object`, pa ih treba pretvoriti: `ToString()` ili eksplicitna konverzija tipa.

### 6.5 DataSet: više tabela u memoriji

```csharp
DataSet ds = new DataSet();

SqlDataAdapter da1 = new SqlDataAdapter("SELECT * FROM Ucenici", con);
SqlDataAdapter da2 = new SqlDataAdapter("SELECT * FROM Odeljenja", con);

da1.Fill(ds, "Ucenici");
da2.Fill(ds, "Odeljenja");

GridView1.DataSource = ds.Tables["Ucenici"];
GridView1.DataBind();
```

- `DataSet` je kolekcija `DataTable` objekata, nešto poput male baze u memoriji.
- `Fill(ds, "Ucenici")` dodaje tabelu pod tim imenom.
- `ds.Tables["Ucenici"]` vraća traženu tabelu.

### 6.6 Izmena podataka u memoriji

```csharp
// izmena vrednosti
dt.Rows[0]["Prezime"] = "Petrović";

// dodavanje novog reda
DataRow nov = dt.NewRow();
nov["Ime"] = "Sara";
nov["Prezime"] = "Lukić";
nov["OdeljenjeId"] = 1;
dt.Rows.Add(nov);

// brisanje reda
dt.Rows[2].Delete();
```

**Sve promene su samo u memoriji. Baza ostaje nepromenjena.**

- `NewRow()` pravi prazan red iste strukture. Kolonu `Id` ne popunjavamo jer je identity.
- `OdeljenjeId` mora imati vrednost koja postoji u tabeli Odeljenja (strani ključ).
- `Delete()` obeležava red za brisanje.

### 6.7 RowState: šta adapter šalje bazi

Svaki `DataRow` pamti svoje stanje. Na osnovu njega adapter bira koju naredbu da pošalje.

| RowState | Značenje | Naredba pri Update |
|---|---|---|
| `Unchanged` | red nije menjan | nema naredbe |
| `Modified` | vrednost je izmenjena | `UPDATE` |
| `Added` | red je dodat | `INSERT` |
| `Deleted` | red je obeležen za brisanje | `DELETE` |

```csharp
dt.Rows[0]["Prezime"] = "Petrović";
// dt.Rows[0].RowState je sada DataRowState.Modified
```

Posle uspešnog poziva `Update` stanja se vraćaju na `Unchanged`.

### 6.8 Update: vraćanje izmena u bazu

```csharp
protected void btnSnimi_Click(object sender, EventArgs e)
{
    SqlConnection con = new SqlConnection();
    con.ConnectionString =
        Properties.Settings.Default.SkolaConnectionString;

    SqlDataAdapter da = new SqlDataAdapter("SELECT * FROM Ucenici", con);
    SqlCommandBuilder cb = new SqlCommandBuilder(da);

    DataTable dt = new DataTable();
    da.Fill(dt);

    dt.Rows[0]["Prezime"] = txtPrezime.Text;   // izmena u memoriji
    da.Update(dt);                              // izmena u bazi

    GridView1.DataSource = dt;
    GridView1.DataBind();
}
```

- `SqlCommandBuilder` sam pravi `INSERT`, `UPDATE` i `DELETE` naredbe na osnovu `SELECT` naredbe.
- `Update` šalje samo izmenjene, dodate i obrisane redove.
- **Uslov:** `SELECT` mora da čita jednu tabelu koja ima primarni ključ.

### 6.9 Web aplikacija: gde čuvati DataTable?

Web stranica se pri svakom zahtevu pravi iznova, pa `DataTable` ne preživljava *postback*. Postoje tri rešenja:

| Rešenje | Opis | Kod |
|---|---|---|
| Ponovo učitati iz baze | najjednostavnije i najsigurnije: `Fill` pri svakom zahtevu | `da.Fill(dt);` |
| `Session` | čuva se na serveru, po korisniku; troši memoriju servera | `Session["dt"] = dt;` |
| `ViewState` | čuva se u samoj stranici; povećava njenu veličinu | `ViewState["dt"] = dt;` |

Čitanje iz Session-a: `DataTable dt = (DataTable)Session["dt"];`

Za vežbu koristi prvu varijantu. Na velikim tabelama `Session` i `ViewState` su loš izbor.

---

## 7. Česte greške

| Šta se dešava | Uzrok i rešenje |
|---|---|
| Izmene se ne vide u bazi | Nedostaje poziv `da.Update(dt)`. |
| `InvalidOperationException` kod `Update` | Tabela nema primarni ključ ili `SELECT` spaja više tabela. |
| GridView je prazan | Nedostaje `DataBind()` ili `DataTable` nije popunjen. |
| Izmene nestaju posle klika | `DataTable` nije sačuvan (`Session`) niti ponovo učitan. |
| *Both DataSource and DataSourceID are defined* | GridView još ima `DataSourceID` iz Primera 1: ukloni ga ili napravi novu stranicu. |
| Greška pri brisanju odeljenja | Odeljenje ima učenike (strani ključ). Obriši prvo učenike ili probaj odeljenje III-4. |
| *Cannot open database* ili greška pri prijavi | Proveri naziv servera i baze u konekcionom stringu (`Data Source`, `Initial Catalog`). |

---

## 8. Zadaci za vežbu

Prva četiri zadatka su obavezna, a peti je dodatni.

1. **Primer 1 u praksi.** Poveži GridView sa tabelom **Odeljenja** pomoću SqlDataSource; uključi izmenu i brisanje.
2. **Konekcioni string i DataTable.** Konekcioni string upiši u Settings i prikaži tabelu **Ucenici** u GridView-u (DataTable).
3. **DataSet.** Napuni DataSet tabelama **Ucenici** i **Odeljenja** i prikaži svaku u zasebnom GridView-u.
4. **Unos novog učenika.** Napravi formu za unos novog učenika i snimi ga u bazu pomoću `Update`.
5. **Za one koji žele više.** Filtriraj prikaz učenika pomoću `DataView` i svojstva `RowFilter`, na primer `dt.DefaultView.RowFilter = "Prezime LIKE 'P%'";`

Savet za zadatak 1: ako želiš da dodaješ nova odeljenja, dodaj na formu kontrolu **DetailsView** i poveži je sa istim SqlDataSource-om.

---

## 9. Rezime i pitanja za ponavljanje

**Rezime**

- SqlDataSource povezuje GridView sa bazom i sam generiše Select, Insert, Update i Delete.
- Beskonekcioni pristup: podaci se preuzmu u memoriju, a veza sa bazom se zatvara.
- SqlDataAdapter povezuje bazu i memoriju: Fill puni, a Update vraća izmene.
- DataTable je jedna tabela, a DataSet je skup tabela u memoriji.
- U web aplikaciji DataTable treba ponovo učitati ili sačuvati (Session, ViewState).

**Pitanja za ponavljanje**

1. Koja je glavna razlika između konekcionog i beskonekcionog pristupa?
2. Šta SqlDataSource generiše umesto nas kada uključimo automatski Insert, Update i Delete?
3. Šta radi metoda `Fill`, a šta metoda `Update`?
4. U čemu je razlika između `DataTable` i `DataSet`?
5. Zašto u web aplikaciji moramo da vodimo računa o tome gde se čuva `DataTable`?
