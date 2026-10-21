# Korisnički nalozi i kontrola pristupa u ASP.NET aplikaciji

**Dodatak za učenike · Web programiranje · Visual Studio 2010, C#, MS SQL Server**

Većina web aplikacija ima stranice koje ne sme da vidi svako: neke su samo za prijavljene korisnike, a neke samo za administratore. U ovom dodatku učimo kako se prave korisnički nalozi, kako se određuju i kontrolišu prava pristupa i kako se koriste kontrole **Login** i **CreateUserWizard** (kontrola Create User).

## Sadržaj

1. Šta ćemo naučiti
2. Osnovni pojmovi: kako radi prijava
3. Priprema: baza za korisničke naloge
4. Pravljenje naloga i uloga
5. Kontrola CreateUserWizard (Create User)
6. Kontrola Login
7. Prava pristupa
8. Česte greške
9. Zadaci za vežbu
10. Rezime i pitanja za ponavljanje

---

## 1. Šta ćemo naučiti

1. Objasniti razliku između autentifikacije i autorizacije.
2. Pripremiti bazu u kojoj se čuvaju korisnici i uloge i podesiti `Web.config`.
3. Napraviti korisničke naloge i uloge alatom ASP.NET Configuration, kontrolom CreateUserWizard i kodom.
4. Napraviti stranicu za prijavu pomoću kontrole Login.
5. Odrediti prava pristupa pravilima u `Web.config` fajlu i proveriti ih u kodu.

---

## 2. Osnovni pojmovi: kako radi prijava

| Pojam | Pitanje na koje odgovara | Primer |
|---|---|---|
| **Autentifikacija** | Ko si ti? | prijava korisničkim imenom i lozinkom |
| **Autorizacija** | Šta smeš da radiš? | stranicu Admin vidi samo administrator |
| **Uloga (role)** | Grupa korisnika sa istim pravima | `Admin`, `Ucenik` |
| **Membership** | ASP.NET servis koji čuva naloge i proverava lozinke | tabele `aspnet_Users`, `aspnet_Membership` |
| **Forms authentication** | Prijava preko web forme; server pamti prijavu u kolačiću (cookie) | stranica `Login.aspx` |

**Šta se dešava kada korisnik otvori zaštićenu stranicu:**

1. Korisnik zatraži stranicu iz zaštićenog foldera.
2. ASP.NET pročita pravila u `Web.config` fajlu. Korisnik nije prijavljen, pa ga preusmerava na `Login.aspx`.
3. Korisnik upiše korisničko ime i lozinku. Kontrola Login proverava ih u bazi (`Membership.ValidateUser`).
4. Ako su podaci tačni, server pravi **kolačić za prijavu** i šalje ga pregledaču.
5. Pregledač šalje kolačić uz svaki sledeći zahtev, pa server zna ko je korisnik (`User.Identity.Name`) i kojim ulogama pripada (`User.IsInRole`).
6. Korisnik se vraća na stranicu koju je prvobitno tražio.

> **Lozinke se ne čuvaju kao običan tekst.** Membership u bazu upisuje samo *heš* lozinke (rezultat jednosmernog izračunavanja) i *so* (salt). Lozinku zato niko ne može da pročita, čak ni administrator baze.

---

## 3. Priprema: baza za korisničke naloge

ASP.NET Membership čuva naloge i uloge u posebnoj bazi. Podrazumevano očekuje SQL Server Express fajl, a mi radimo na punom SQL Server-u, pa bazu pravimo alatom **aspnet_regsql**, a u `Web.config` upisujemo njen konekcioni string.

### 3.1 Pravljenje baze alatom aspnet_regsql

1. Otvori **Command Prompt** (komandna linija).
2. Pređi u folder sa .NET Framework 4 alatima:

   ```text
   cd C:\Windows\Microsoft.NET\Framework\v4.0.30319
   ```

   Na 64-bitnom Windows-u može se koristiti i folder `C:\Windows\Microsoft.NET\Framework64\v4.0.30319`.
3. Pokreni čarobnjak:

   ```text
   aspnet_regsql
   ```

4. U čarobnjaku:
   - **Welcome** › **Next**
   - izaberi **Configure SQL Server for application services** › **Next**
   - **Server**: upiši naziv svog SQL Server-a (isti kao u Management Studiju)
   - izaberi **Windows authentication**
   - **Database**: ostavi `<default>`, čime se pravi baza **aspnetdb**
   - **Next** › **Next** › **Finish**
5. U Management Studiju proveri da postoji baza **aspnetdb** sa tabelama `aspnet_Users`, `aspnet_Membership`, `aspnet_Roles`, `aspnet_UsersInRoles` i drugim.

Isto se može uraditi jednom naredbom (`-E` je Windows autentifikacija, `-A mr` dodaje podršku za *membership* i *roles*):

```text
aspnet_regsql.exe -S SERVER -E -A mr
```

### 3.2 Podešavanje Web.config fajla

Otvori `Web.config` projekta. Odeljak `connectionStrings` treba da sadrži vezu ka bazi aspnetdb pod nazivom `ApplicationServices`, a u odeljku `system.web` podešavamo prijavu, membership i uloge.

**Konekcioni string** (zameni `SERVER` nazivom svog SQL Server-a):

```xml
<connectionStrings>
  <add name="ApplicationServices"
       connectionString="Data Source=SERVER;Initial Catalog=aspnetdb;Integrated Security=True"
       providerName="System.Data.SqlClient" />
</connectionStrings>
```

> Ako je u fajlu već postojao `ApplicationServices` (šablon projekta ga pravi za SQL Express), **zameni ga**, nemoj dodavati drugi sa istim nazivom.

**Prijava, membership i uloge** (unutar `<system.web>`):

```xml
<authentication mode="Forms">
  <forms loginUrl="~/Login.aspx" defaultUrl="~/Default.aspx" timeout="30" />
</authentication>

<membership defaultProvider="AspNetSqlMembershipProvider">
  <providers>
    <clear />
    <add name="AspNetSqlMembershipProvider"
         type="System.Web.Security.SqlMembershipProvider"
         connectionStringName="ApplicationServices"
         applicationName="/"
         enablePasswordRetrieval="false"
         enablePasswordReset="true"
         requiresQuestionAndAnswer="false"
         requiresUniqueEmail="false"
         minRequiredPasswordLength="6"
         minRequiredNonalphanumericCharacters="0"
         maxInvalidPasswordAttempts="5"
         passwordAttemptWindow="10" />
  </providers>
</membership>

<roleManager enabled="true" defaultProvider="AspNetSqlRoleProvider">
  <providers>
    <clear />
    <add name="AspNetSqlRoleProvider"
         type="System.Web.Security.SqlRoleProvider"
         connectionStringName="ApplicationServices"
         applicationName="/" />
  </providers>
</roleManager>
```

| Podešavanje | Značenje |
|---|---|
| `authentication mode="Forms"` | prijava preko web forme |
| `loginUrl` | stranica na koju se preusmeravaju neprijavljeni korisnici |
| `<clear />` | briše podrazumevane provajdere, da se koristi samo naš |
| `connectionStringName` | koji konekcioni string provajder koristi |
| `applicationName` | aplikacija kojoj nalozi pripadaju (mora biti isto za membership i roleManager) |
| `minRequiredPasswordLength` | najmanja dužina lozinke |
| `minRequiredNonalphanumericCharacters` | koliko znakova koji nisu slova ni cifre lozinka mora da ima |
| `maxInvalidPasswordAttempts`, `passwordAttemptWindow` | posle 5 pogrešnih pokušaja u 10 minuta nalog se zaključava |
| `roleManager enabled="true"` | uključuje uloge |

> Ako koristiš šablon **ASP.NET Web Application**, projekat već ima stranice u folderu `Account`, pa je `loginUrl` tamo `~/Account/Login.aspx`. U ovom dodatku stranice pravimo sami, u korenu projekta.
>
> Na razvojnom serveru Visual Studija 2010 aplikacija radi pod tvojim Windows nalogom, pa `Integrated Security=True` radi bez dodatnog podešavanja. Na pravom IIS serveru nalog pod kojim radi aplikacija mora imati pristup bazi aspnetdb.

### 3.3 Struktura projekta

Napravi sledeće stranice i foldere (desni klik na projekat › **Add**):

```text
Default.aspx          početna stranica
Login.aspx            stranica za prijavu (kontrola Login)
Register.aspx         pravljenje naloga (kontrola CreateUserWizard)
Korisnici/
    Default.aspx      stranica za sve prijavljene korisnike
Admin/
    Default.aspx      stranica samo za administratore
```

---

## 4. Pravljenje naloga i uloga

Nalozi se mogu praviti na tri načina: alatom ASP.NET Configuration, kontrolom CreateUserWizard (odeljak 5) i kodom (odeljak 4.3).

### 4.1 Alat ASP.NET Configuration (Web Site Administration Tool)

1. Izaberi **Project › ASP.NET Configuration** (kod projekta tipa Web Site: **Website › ASP.NET Configuration**). Alat se otvara u pregledaču.
2. Klikni karticu **Security**. Ako te alat pita za vrstu autentifikacije, izaberi **From the internet** (Forms).
3. **Uloge:** klikni **Create or manage roles**. Napravi dve uloge: `Admin` i `Ucenik` (**Add Role**).
4. **Korisnici:** klikni **Create user** i popuni polja:

   | Polje | Primer |
   |---|---|
   | User Name | `nastavnik` |
   | Password / Confirm Password | `lozinka123` (mora ispuniti pravila iz Web.config-a) |
   | E-mail | `nastavnik@example.com` |
   | Active User | označeno |
   | Roles | označi `Admin` |

   Isto tako napravi korisnika `ucenik1` sa ulogom `Ucenik`.
5. **Upravljanje:** **Manage users** omogućava izmenu uloga (**Edit roles**), brisanje naloga i deaktiviranje (poništi **Active User**).
6. **Pravila pristupa:** **Create access rules** pravi pravila za izabrani folder (vidi odeljak 7.4).

Ako alat prijavi grešku pri povezivanju sa bazom (*Unable to connect to SQL Server database*), proveri konekcioni string `ApplicationServices` u `Web.config` fajlu (odeljak 8).

### 4.2 Šta je upisano u bazu

U Management Studiju otvori bazu **aspnetdb** i pokreni upite:

```sql
USE aspnetdb;

-- svi korisnici
SELECT UserName, LastActivityDate FROM aspnet_Users;

-- podaci o nalozima (lozinka je heš, ne pravi tekst)
SELECT u.UserName, m.Email, m.IsApproved, m.IsLockedOut, m.CreateDate
FROM aspnet_Users u
JOIN aspnet_Membership m ON m.UserId = u.UserId;

-- uloge
SELECT RoleName FROM aspnet_Roles;

-- ko ima koju ulogu
SELECT u.UserName, r.RoleName
FROM aspnet_UsersInRoles ur
JOIN aspnet_Users u ON u.UserId = ur.UserId
JOIN aspnet_Roles r ON r.RoleId = ur.RoleId;
```

Obrati pažnju na kolone `Password` i `PasswordSalt` u tabeli `aspnet_Membership`: tamo se ne vidi tvoja lozinka, nego niz nasumičnih znakova.

### 4.3 Pravljenje naloga i uloga kodom

Klasa `Membership` radi sa nalozima, a klasa `Roles` sa ulogama (prostor imena `System.Web.Security`).

```csharp
using System.Web.Security;

// uloga
if (!Roles.RoleExists("Ucenik"))
    Roles.CreateRole("Ucenik");

// nalog
MembershipCreateStatus status;
Membership.CreateUser("ucenik2", "lozinka123", "ucenik2@example.com",
                      null, null, true, out status);

if (status == MembershipCreateStatus.Success)
    Roles.AddUserToRole("ucenik2", "Ucenik");
else
    Label1.Text = "Greška: " + status;   // na primer DuplicateUserName, InvalidPassword
```

| Metoda | Šta radi |
|---|---|
| `Membership.CreateUser(...)` | pravi nalog |
| `Membership.ValidateUser(ime, lozinka)` | proverava korisničko ime i lozinku |
| `Membership.GetUser(ime)` | vraća podatke o nalogu (`null` ako ne postoji) |
| `Membership.GetAllUsers()` | vraća sve naloge |
| `Membership.DeleteUser(ime)` | briše nalog |
| `Roles.CreateRole(uloga)` | pravi ulogu |
| `Roles.AddUserToRole(ime, uloga)` | dodaje korisnika u ulogu |
| `Roles.GetRolesForUser(ime)` | vraća uloge korisnika |

Spisak svih naloga u GridView-u:

```csharp
GridView1.DataSource = Membership.GetAllUsers();
GridView1.DataBind();
```

---

## 5. Kontrola CreateUserWizard (Create User)

Kontrola **CreateUserWizard** je gotova forma za registraciju. Sama proverava podatke, pravi nalog u bazi i prikazuje poruku o uspehu. Nalazi se u Toolbox-u, u grupi **Login**.

### 5.1 Stranica Register.aspx

1. Otvori `Register.aspx` i prevuci **CreateUserWizard** na formu.
2. U prozoru **Properties** podesi `ContinueDestinationPageUrl` na `~/Default.aspx`.
3. Pokreni stranicu (**Ctrl+F5**) i napravi nalog. Posle uspeha pojavi se poruka *Your account has been successfully created*, a dugme **Continue** vodi na početnu stranicu.
4. Proveri nalog u bazi upitom iz odeljka 4.2.

```html
<asp:CreateUserWizard ID="CreateUserWizard1" runat="server"
    ContinueDestinationPageUrl="~/Default.aspx">
</asp:CreateUserWizard>
```

Kontrola ima dva koraka: **CreateUserWizardStep** (unos podataka) i **CompleteWizardStep** (poruka o uspehu).

### 5.2 Važna svojstva

| Svojstvo | Značenje |
|---|---|
| `ContinueDestinationPageUrl` | stranica na koju vodi dugme Continue |
| `LoginCreatedUser` | da li se novi korisnik odmah prijavljuje (podrazumevano `True`) |
| `RequireEmail` | da li je e-mail obavezan (podrazumevano `True`) |
| `DisableCreatedUser` | novi nalog je neaktivan dok ga administrator ne aktivira |
| `CompleteSuccessText` | poruka posle uspešnog pravljenja naloga |
| `DuplicateUserNameErrorMessage` | poruka kada korisničko ime već postoji |
| `InvalidPasswordErrorMessage` | poruka kada lozinka ne ispunjava pravila |
| `UserName`, `Email`, `Password` | vrednosti koje je korisnik upisao (čitaju se u događajima) |

Poruke možeš da prevedeš na srpski, na primer:

```html
<asp:CreateUserWizard ID="CreateUserWizard1" runat="server"
    ContinueDestinationPageUrl="~/Default.aspx"
    CompleteSuccessText="Nalog je uspešno napravljen."
    DuplicateUserNameErrorMessage="To korisničko ime je zauzeto."
    InvalidPasswordErrorMessage="Lozinka mora imati najmanje 6 znakova.">
</asp:CreateUserWizard>
```

### 5.3 Događaji

| Događaj | Kada se javlja |
|---|---|
| `CreatingUser` | pre pravljenja naloga |
| `CreatedUser` | posle uspešnog pravljenja naloga |
| `CreateUserError` | kada pravljenje naloga ne uspe |
| `ContinueButtonClick` | kada korisnik klikne Continue |

**Primer: svaki novi korisnik automatski postaje učenik.** Dvaput klikni na kontrolu u Design prikazu (ili u Properties izaberi događaj `CreatedUser`) i upiši:

```csharp
protected void CreateUserWizard1_CreatedUser(object sender, EventArgs e)
{
    if (!Roles.RoleExists("Ucenik"))
        Roles.CreateRole("Ucenik");

    Roles.AddUserToRole(CreateUserWizard1.UserName, "Ucenik");
}
```

U markupu kontrole dodaje se `OnCreatedUser="CreateUserWizard1_CreatedUser"`.

> Ako ne želiš da se novi korisnik odmah prijavi, postavi `LoginCreatedUser="False"`.

---

## 6. Kontrola Login

Kontrola **Login** je gotova forma za prijavu sa poljima za korisničko ime i lozinku i dugmetom. Sama poziva `Membership.ValidateUser`, a kod uspešne prijave pravi kolačić.

### 6.1 Stranica Login.aspx

1. Otvori `Login.aspx` i prevuci **Login** na formu.
2. Podesi svojstva (primer ispod).
3. Pokreni stranicu i prijavi se nalogom koji si napravio.

```html
<asp:Login ID="Login1" runat="server"
    DestinationPageUrl="~/Default.aspx"
    CreateUserText="Nemaš nalog? Registruj se"
    CreateUserUrl="~/Register.aspx"
    TitleText="Prijava"
    UserNameLabelText="Korisničko ime:"
    PasswordLabelText="Lozinka:"
    LoginButtonText="Prijavi se"
    RememberMeText="Zapamti me"
    FailureText="Pogrešno korisničko ime ili lozinka.">
</asp:Login>
```

### 6.2 Važna svojstva

| Svojstvo | Značenje |
|---|---|
| `DestinationPageUrl` | gde se korisnik prebacuje posle prijave, ako je Login otvoren direktno |
| `CreateUserText`, `CreateUserUrl` | link za registraciju (vodi na `Register.aspx`) |
| `PasswordRecoveryText`, `PasswordRecoveryUrl` | link za zaboravljenu lozinku |
| `DisplayRememberMe` | prikazuje polje *Zapamti me* |
| `FailureText` | poruka kada prijava ne uspe |
| `TitleText`, `UserNameLabelText`, `PasswordLabelText`, `LoginButtonText` | tekstovi na formi |

> **Povratak na traženu stranicu.** Ako je korisnik na `Login.aspx` stigao zato što je tražio zaštićenu stranicu, adresa u pregledaču sadrži `?ReturnUrl=...`. Posle prijave korisnik se vraća na **tu** stranicu, a `DestinationPageUrl` se koristi samo kada je Login otvoren direktno.

### 6.3 Događaji

| Događaj | Kada se javlja |
|---|---|
| `LoggingIn` | pre provere podataka |
| `Authenticate` | pri proveri podataka (može da se zameni sopstvenom proverom) |
| `LoggedIn` | posle uspešne prijave |
| `LoginError` | kada prijava ne uspe |

**Primer: administrator se posle prijave šalje na svoju stranicu.**

```csharp
protected void Login1_LoggedIn(object sender, EventArgs e)
{
    if (Roles.IsUserInRole(Login1.UserName, "Admin"))
        Response.Redirect("~/Admin/Default.aspx");
}
```

U događaju `LoggedIn` prijava još nije završena za tekući zahtev, pa se uloga proverava preko `Roles.IsUserInRole(ime, uloga)`, a ne preko `User.IsInRole`.

### 6.4 Zaključavanje naloga

Posle 5 pogrešnih lozinki u 10 minuta nalog se zaključava (`maxInvalidPasswordAttempts`, `passwordAttemptWindow`). Otključava se kodom:

```csharp
MembershipUser korisnik = Membership.GetUser("ucenik1");
korisnik.UnlockUser();
```

ili upitom u bazi aspnetdb:

```sql
UPDATE aspnet_Membership SET IsLockedOut = 0 WHERE UserId IN
    (SELECT UserId FROM aspnet_Users WHERE UserName = 'ucenik1');
```

### 6.5 Pomoćne kontrole iz grupe Login

| Kontrola | Šta radi |
|---|---|
| **LoginStatus** | prikazuje link *Prijavi se* ili *Odjavi se*, zavisno od toga da li je korisnik prijavljen |
| **LoginName** | prikazuje korisničko ime prijavljenog korisnika |
| **LoginView** | prikazuje različit sadržaj za neprijavljene, prijavljene i korisnike određene uloge |
| **ChangePassword** | forma za promenu lozinke |
| **PasswordRecovery** | oporavak zaboravljene lozinke (traži podešeno slanje e-pošte) |

**Primer za Default.aspx:**

```html
<asp:LoginView ID="LoginView1" runat="server">
    <AnonymousTemplate>
        Nisi prijavljen. <a href="Login.aspx">Prijavi se</a>
    </AnonymousTemplate>
    <LoggedInTemplate>
        Dobrodošao, <asp:LoginName ID="LoginName1" runat="server" />!
    </LoggedInTemplate>
    <RoleGroups>
        <asp:RoleGroup Roles="Admin">
            <ContentTemplate>
                Dobrodošao, <asp:LoginName ID="LoginName2" runat="server" />!
                Imaš administratorska prava: <a href="Admin/Default.aspx">Admin stranica</a>
            </ContentTemplate>
        </asp:RoleGroup>
    </RoleGroups>
</asp:LoginView>

<asp:LoginStatus ID="LoginStatus1" runat="server"
    LoginText="Prijavi se" LogoutText="Odjavi se"
    LogoutAction="Redirect" LogoutPageUrl="~/Default.aspx" />
```

Ako korisnik pripada ulozi iz `RoleGroups`, prikazuje se sadržaj te grupe **umesto** sadržaja iz `LoggedInTemplate`.

Odjava iz koda: `FormsAuthentication.SignOut();`

---

## 7. Prava pristupa

Prava pristupa određujemo pravilima `<authorization>` u `Web.config` fajlu. Pravilo se odnosi na folder u kome se fajl nalazi i na sve podfoldere.

### 7.1 Pravila allow i deny

| Oznaka | Na koga se odnosi |
|---|---|
| `users="?"` | anonimni (neprijavljeni) korisnici |
| `users="*"` | svi korisnici |
| `users="ucenik1"` | jedan korisnik |
| `roles="Admin"` | svi korisnici sa ulogom Admin |

**Pravila se čitaju odozgo nadole i važi prvo pravilo koje odgovara korisniku.** Zato redosled mora da bude ovakav:

```xml
<allow roles="Admin" />
<deny users="*" />
```

Ako se okrenu mesta (`deny users="*"` pa tek onda `allow`), niko ne dobija pristup, pa ni Admin.

### 7.2 Folder dostupan samo prijavljenim korisnicima

U folder `Korisnici` dodaj novi fajl `Web.config` (**Add › New Item › Web Configuration File**) i zameni njegov sadržaj ovim:

```xml
<?xml version="1.0"?>
<configuration>
  <system.web>
    <authorization>
      <deny users="?" />
    </authorization>
  </system.web>
</configuration>
```

Anonimni korisnik koji pokuša da otvori `Korisnici/Default.aspx` biva preusmeren na `Login.aspx`.

### 7.3 Folder dostupan samo administratorima

U folder `Admin` dodaj `Web.config`:

```xml
<?xml version="1.0"?>
<configuration>
  <system.web>
    <authorization>
      <allow roles="Admin" />
      <deny users="*" />
    </authorization>
  </system.web>
</configuration>
```

### 7.4 Pravila pomoću alata ASP.NET Configuration

Isti fajlovi mogu se napraviti bez ručnog pisanja: **ASP.NET Configuration › Security › Create access rules**.

1. Izaberi folder (na primer `Admin`).
2. **Rule applies to**: izaberi *Role* (i uloga `Admin`), *User*, *All users* ili *Anonymous users*.
3. **Permission**: *Allow* ili *Deny*.
4. Klikni **OK**. Dodaj sledeće pravilo (na primer *All users › Deny*).

Alat upisuje pravila u `Web.config` tog foldera. Pravila se mogu videti i menjati u **Manage access rules**. Posle toga osveži Solution Explorer da se pojavi novi fajl.

### 7.5 Cela aplikacija zaštićena, osim izabranih stranica

U `Web.config` u korenu projekta pravilo `authorization` ide u postojeći odeljak `<system.web>`, a blok `<location>` ide u `<configuration>`, van `<system.web>`:

```xml
<configuration>
  <system.web>
    <authorization>
      <deny users="?" />
    </authorization>
    <!-- ostala podešavanja iz odeljka 3.2 -->
  </system.web>

  <location path="Register.aspx">
    <system.web>
      <authorization>
        <allow users="*" />
      </authorization>
    </system.web>
  </location>
</configuration>
```

Sada svaka stranica traži prijavu, osim `Register.aspx`, jer mora biti dostupna novim korisnicima. Stranica `Login.aspx` je uvek dostupna.

Ako stranica za prijavu izgleda bez stilova, i fajl sa stilovima je zaštićen. Dodaj isti `<location>` blok za njegovu putanju, na primer `path="Styles"`.

### 7.6 Provera prava u kodu

Pravila u `Web.config` fajlu štite stranice. Kodom možemo da prilagodimo sadržaj stranice.

| Izraz | Značenje |
|---|---|
| `User.Identity.IsAuthenticated` | `true` ako je korisnik prijavljen |
| `User.Identity.Name` | korisničko ime prijavljenog korisnika |
| `User.IsInRole("Admin")` | `true` ako prijavljeni korisnik ima ulogu |
| `Roles.GetRolesForUser()` | sve uloge prijavljenog korisnika |
| `Membership.GetUser()` | podaci o prijavljenom korisniku (`null` ako nije prijavljen) |

```csharp
protected void Page_Load(object sender, EventArgs e)
{
    if (User.Identity.IsAuthenticated)
        lblPoruka.Text = "Prijavljen si kao " + User.Identity.Name;
    else
        lblPoruka.Text = "Nisi prijavljen.";
}
```

**Veza sa prethodnom lekcijom:** dugmad *Edit* i *Delete* u GridView-u sa tabelom Ucenici (kolona `CommandField` je prva kolona) vidi samo administrator:

```csharp
protected void Page_Load(object sender, EventArgs e)
{
    GridView1.Columns[0].Visible = User.IsInRole("Admin");
}
```

> **Pazi:** sakrivanje dugmadi nije prava zaštita. Stranice koje menjaju podatke stavi u zaštićen folder (odeljak 7.3), a kod koristi samo za prilagođavanje izgleda.

### 7.7 Plan provere

Kada završiš, proveri da li sve radi kao što piše u tabeli:

| Korisnik | `Default.aspx` | `Korisnici/Default.aspx` | `Admin/Default.aspx` |
|---|---|---|---|
| neprijavljen | otvara se | prelazi na Login | prelazi na Login |
| `ucenik1` (Ucenik) | otvara se | otvara se | prelazi na Login |
| `nastavnik` (Admin) | otvara se | otvara se | otvara se |

---

## 8. Česte greške

| Šta se dešava | Uzrok i rešenje |
|---|---|
| *Unable to connect to SQL Server database* ili *Cannot open database "aspnetdb"* | Baza nije napravljena alatom aspnet_regsql ili je pogrešan naziv servera u konekcionom stringu `ApplicationServices`. |
| Nalozi se prave u pogrešnoj bazi ili se javlja greška o `.mdf` fajlu | U `Web.config` je ostao stari `ApplicationServices` za SQL Express. Zameni ga stringom iz odeljka 3.2. |
| *The Role Manager feature has not been enabled* | Nedostaje `<roleManager enabled="true">`. |
| Lozinka se odbija | Lozinka ne ispunjava pravila (`minRequiredPasswordLength`, `minRequiredNonalphanumericCharacters`). |
| Posle prijave se ništa ne dešava ili se pojavljuje greška o `Login.aspx` | Nedostaje `<authentication mode="Forms">` ili `loginUrl` ne pokazuje na pravu stranicu. |
| Niko ne može da uđe u folder `Admin`, ni administrator | Pravila su u pogrešnom redosledu: `allow` mora biti ispred `deny users="*"`. |
| `Register.aspx` traži prijavu | Cela aplikacija je zaštićena sa `deny users="?"`, a za `Register.aspx` nema bloka `<location>` (odeljak 7.5). |
| Stranica za prijavu je bez stilova | Fajl sa stilovima je u zaštićenom folderu: dodaj `<location>` sa dozvolom za taj folder. |
| Nalog je zaključan (`IsLockedOut = 1`) | Bilo je 5 pogrešnih pokušaja prijave. Otključaj nalog (odeljak 6.4). |
| Kontrola CreateUserWizard traži e-mail ili sigurnosno pitanje | `RequireEmail="True"` je podrazumevano; sigurnosno pitanje traži provajder ako je `requiresQuestionAndAnswer="true"`. |
| Greška *DuplicateUserName* | Korisničko ime već postoji. |
| `User.IsInRole` vraća `false` u događaju `LoggedIn` | Koristi `Roles.IsUserInRole(Login1.UserName, "Admin")`. |

---

## 9. Zadaci za vežbu

1. **Priprema.** Napravi bazu aspnetdb alatom aspnet_regsql i podesi `Web.config` (odeljak 3). Proveri u alatu ASP.NET Configuration da se kartica Security otvara bez greške.
2. **Uloge i nalozi.** Napravi uloge `Admin` i `Ucenik`, korisnika `nastavnik` (Admin) i korisnika `ucenik1` (Ucenik). U Management Studiju pokreni upit koji prikazuje ko ima koju ulogu.
3. **Registracija.** Napravi `Register.aspx` sa kontrolom CreateUserWizard. Poruke prevedi na srpski, a svaki novi korisnik neka automatski dobije ulogu `Ucenik`.
4. **Prijava.** Napravi `Login.aspx` sa kontrolom Login (srpski tekstovi, link ka registraciji). Na `Default.aspx` dodaj LoginView, LoginName i LoginStatus: neprijavljen korisnik vidi poziv za prijavu, prijavljen vidi pozdrav, a administrator i link ka administratorskoj stranici.
5. **Prava pristupa.** Napravi foldere `Korisnici` i `Admin` sa odgovarajućim `Web.config` fajlovima i popuni tabelu provere iz odeljka 7.7.
6. **Povezivanje sa bazom skola21oktobar.** Na stranicu u folderu `Korisnici` stavi GridView sa tabelom Ucenici iz prethodne lekcije. Dugmad za izmenu i brisanje neka vidi samo administrator.

**Za one koji žele više:**

- Dodaj stranicu u folderu `Admin` sa GridView-om koji prikazuje sve naloge (`Membership.GetAllUsers()`) i dugmetom za brisanje izabranog naloga (`Membership.DeleteUser`).
- Dodaj stranicu sa kontrolom ChangePassword za prijavljene korisnike.

---

## 10. Rezime i pitanja za ponavljanje

**Rezime**

- Autentifikacija proverava ko je korisnik, a autorizacija određuje šta sme da radi.
- Nalozi, lozinke (kao heš) i uloge čuvaju se u bazi koju pravi alat aspnet_regsql, a povezuje je konekcioni string `ApplicationServices` u `Web.config` fajlu.
- Naloge i uloge možemo praviti alatom ASP.NET Configuration, kontrolom CreateUserWizard ili kodom (`Membership`, `Roles`).
- Kontrola Login proverava korisničko ime i lozinku i pravi kolačić za prijavu.
- Prava pristupa određuju pravila `allow` i `deny` u `Web.config` fajlu foldera. Važi prvo pravilo koje odgovara korisniku.
- U kodu se prava proveravaju sa `User.Identity.IsAuthenticated` i `User.IsInRole`.

**Pitanja za ponavljanje**

1. Koja je razlika između autentifikacije i autorizacije?
2. Zašto se lozinke u bazi čuvaju kao heš, a ne kao običan tekst?
3. Šta znače oznake `?` i `*` u pravilima `allow` i `deny`?
4. Zašto je redosled pravila `allow roles="Admin"` i `deny users="*"` bitan?
5. Šta radi kontrola Login, a šta kontrola CreateUserWizard?
6. Kako bi iz koda proverio da li prijavljeni korisnik ima ulogu `Admin`?
