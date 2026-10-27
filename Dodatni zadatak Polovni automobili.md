# Dodatni zadatak: Polovni automobili

Oct 28, 2026 · @Vanja

## Šta treba napraviti

Napravi stranicu `Vozila.aspx` za oglasnik polovnih automobila. Korisnik bira proizvođača, pa model, i vidi sva vozila tog modela. Zatim može da doda novo vozilo, izmeni pređene kilometre izabranog vozila ili ga obriše.

Zadatak se radi samostalno, po uzoru na lekciju **ListBox i DropDownList: prikaz i izmena baze podataka**. Veći deo posla je isti kao u lekciji. Četiri detalja su nova i objašnjena su u delu Uputstva.

| Kontrola | ID | Šta prikazuje ili radi |
| --- | --- | --- |
| DropDownList | `ddlProizvodjaci` | svi proizvođači po nazivu, uvodna stavka „-- izaberite proizvođača --“ sa vrednošću 0 |
| ListBox | `lstModeli` | modeli izabranog proizvođača po nazivu, `Rows = 6` |
| ListBox | `lstVozila` | vozila izabranog modela, jedan red po vozilu, npr. `2021, 12989 km, Siva, Dizel`; `Rows = 10` |
| TextBox | `txtGodina` | godina proizvodnje novog vozila |
| TextBox | `txtKM` | pređeni kilometri, za novo vozilo i za izmenu |
| DropDownList | `ddlBoje` | sve boje, uvodna stavka „-- boja --“ sa vrednošću 0 |
| DropDownList | `ddlGoriva` | sva goriva, uvodna stavka „-- gorivo --“ sa vrednošću 0 |
| Button | `btnDodaj`, `btnIzmeni`, `btnObrisi` | dodaje, menja i briše vozilo |
| Label | `lblPoruka` | javlja šta se desilo, `EnableViewState = False` |

## Baza april\_PolovniAutomobili

Baza ima pet tabela. Vozilo je glavna tabela: svako vozilo pokazuje na jedan model, jednu boju i jedno gorivo, a model pokazuje na proizvođača.

&#91;embedded content: baza april\_PolovniAutomobili · 5 tabela, 4 veze\]

Šta treba znati o podacima:

- Nijedna tabela nema Identity. ID se upisuje ručno, i u skripti i u tvojoj naredbi INSERT.
- U tabeli `Vozilo` je 999 vozila. ID 58 ne postoji, pa je najveći ID 1000.
- 34 modela nemaju nijedno vozilo, na primer Opel Mokka i Ford Kuga.
- Kolona `Registracija` je prazna (NULL) za sva vozila.

Baza se pravi skriptom **A06 Polovni automobili.txt** u SQL Server Management Studiju. Prva linija skripte (`CREATE DATABASE`) je pod komentarom: ako baza ne postoji, prvo pokreni tu liniju bez `--`, pa celu skriptu. Ako baza već postoji na serveru, ovaj korak preskoči. Konekcioni string nazovi `PolovniConnectionString`.

## Zahtevi

### Deo A: prikaz (osnovni)

1. `ddlProizvodjaci` prikazuje sve proizvođače iz tabele `Proizvodjac`, sortirane po nazivu, sa uvodnom stavkom.
2. `lstModeli` prikazuje modele izabranog proizvođača. Kad je izabrana uvodna stavka, lista je prazna.
3. `lstVozila` prikazuje vozila izabranog modela. Svaki red sadrži godinu, pređene kilometre, boju i gorivo, a novija vozila su na vrhu.
4. Kad korisnik promeni proizvođača, lista vozila mora da se isprazni. U njoj ne smeju da ostanu vozila prethodnog modela.
5. `ddlBoje` i `ddlGoriva` prikazuju sve boje i sva goriva, sa uvodnom stavkom.

### Deo B: izmena i brisanje (srednji)

6. **Izmeni** menja pređene kilometre izabranog vozila na vrednost iz `txtKM`. Pre izmene proveri da je vozilo izabrano i da je u `txtKM` ceo broj, 0 ili veći.
7. **Obriši** briše izabrano vozilo, posle potvrde u pregledaču (`confirm`). Pre brisanja proveri da je vozilo izabrano.
8. Posle svake izmene lista vozila prikazuje nove podatke, a `lblPoruka` javlja šta se desilo.

### Deo C: dodavanje (napredni)

9. **Dodaj** dodaje novo vozilo za izabrani model, sa godinom iz `txtGodina`, kilometrima iz `txtKM`, bojom i gorivom iz padajućih lista.
10. Pre dodavanja proveri: model je izabran, godina je ceo broj od 1950 do tekuće godine, kilometri su ceo broj 0 ili veći, boja i gorivo su izabrani.
11. Tabela `Vozilo` nema Identity, pa novi `VoziloID` mora da izračuna sama naredba INSERT.

## Uputstva

Ovo je novo u odnosu na lekciju. Sve ostalo (čarobnjak, `ControlParameter`, naredbe u Command and Parameter Editor-u, metode za obradu događaja) radi se isto kao sa državama.

### 1. Tekst iz više kolona

U čarobnjaku za `lstVozila` izaberi **Specify a custom SQL statement or stored procedure** i sam napiši SELECT. Tekst za prikaz sastavi u upitu kao novu kolonu sa imenom (alias) i nju postavi za `DataTextField`. Brojeve pre spajanja sa tekstom pretvori u tekst sa `CAST`. Dopuni upit:

```sql
SELECT v.VoziloID,
       CAST(v.GodinaProizvodnje AS nvarchar(4)) + N', ' + ... AS Opis
FROM Vozilo v
JOIN Boja b ON v.BojaID = b.BojaID
JOIN ...
WHERE v.ModelID = @ModelID
ORDER BY ...
```

U koraku Define Parameters za `ModelID` podesi Parameter source = Control i ControlID = `lstModeli`.

### 2. Kad u ListBox-u ništa nije izabrano

Tada je `SelectedValue` prazan string, a parametar dobija NULL. SqlDataSource tada uopšte ne izvrši upit, pa lista vozila zadrži stare stavke. Zato u koraku Define Parameters parametru `ModelID` upiši DefaultValue = `0`. Upit se izvrši, ne vrati nijedan red i lista se isprazni.

### 3. Promena proizvođača

SqlDataSource vozila čita izbor modela pre nego što se lista modela napuni iznova, pa još vidi stari model. Napravi metodu za obradu događaja `SelectedIndexChanged` na `ddlProizvodjaci` i u njoj poništi izbor modela:

```csharp
protected void ddlProizvodjaci_SelectedIndexChanged(object sender, EventArgs e)
{
    lstModeli.ClearSelection();
}
```

### 4. Novi ID bez Identity

INSERT može da uzme vrednosti iz SELECT-a umesto iz VALUES. Najveći postojeći ID uvećan za 1 daje `ISNULL(MAX(...), 0) + 1`. Primer za tabelu `Boja`, koji treba prilagoditi tabeli `Vozilo`:

```sql
INSERT INTO Boja (BojaID, NazivBoje)
SELECT ISNULL(MAX(BojaID), 0) + 1, @NazivBoje
FROM Boja
```

Ne koristi `COUNT(*) + 1`. U tabeli je 999 vozila, a najveći ID je 1000 jer ID 58 ne postoji, pa bi `COUNT(*) + 1` dao ID koji je već zauzet.

### 5. Provera da je unet broj

`int.TryParse` vraća `false` ako tekst nije ceo broj, a broj upisuje u promenljivu. Tekuća godina je `DateTime.Now.Year`.

```csharp
int km;
if (!int.TryParse(txtKM.Text, out km) || km < 0)
{
    lblPoruka.Text = "Kilometri moraju biti ceo broj, 0 ili veći.";
    return;
}
```

## Provera rada

Očekivani rezultati važe za bazu napravljenu iz skripte, pre tvojih izmena.

- [ ] Izaberi Volkswagen. U listi modela je 6 modela, od Golf 5 do Passat B8.
- [ ] Izaberi Golf 5. U listi vozila je 27 vozila, a prvo je `2021, 12989 km, Siva, Dizel`.
- [ ] Izaberi Opel, pa Mokka. Lista vozila je prazna, jer taj model nema nijedno vozilo.
- [ ] Izaberi Volkswagen, Golf 5 i jedno vozilo, pa promeni proizvođača na Audi. Lista vozila je prazna.
- [ ] Za Golf 5 dodaj vozilo 2020, 50000 km, Bela, Benzin. Pojavljuje se u listi, a u SQL Server Management Studiju upit `SELECT * FROM Vozilo WHERE VoziloID = 1001` vraća to vozilo.
- [ ] Izaberi novo vozilo, upiši 55000 u `txtKM` i klikni Izmeni. U listi piše `55000 km`.
- [ ] Obriši novo vozilo: prvo Cancel (ništa se ne briše), pa OK (vozilo nestaje).
- [ ] Isprobaj greške: slova u `txtKM`, godina 1900, Dodaj bez boje, Izmeni bez izabranog vozila. Svaki put treba da se pojavi poruka, a ne greška.

## Dodatno za radoznale

1. Dodaj DetailsView koji prikazuje sve podatke izabranog vozila: proizvođača, model, godinu, kilometre, boju i gorivo. Na `lstVozila` uključi AutoPostBack. Posle izmene kilometara DetailsView prikazuje staru vrednost. Objasni zašto i popravi to.
2. Ispod liste vozila prikaži koliko vozila ima izabrani model.
3. Kolona `Registracija` je prazna za sva vozila. Dodaj TextBox za registraciju pri dodavanju novog vozila i prikaži je u listi kad postoji.
