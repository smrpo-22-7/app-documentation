# Prijava v sistem

Kazalo vsebine
1. [Uporaba](#uporaba)
2. [Opis implementacije](#opis-implementacije)
    1. [Hramba poverilnic](#hramba-poverilnic)
    2. [Podprti tokovi](#podprti-tokovi-ang-flows-avtentikacije)
    3. [Dodatni tokovi](#dodatni-tokovi)
    4. [Podpisovanje žetonov](#podpisovanje-žetonov)

## Uporaba

### Prijava

Uporabnik sproži postopek prijave tako, da izbere gumb za prijavo:

![](../media/login_1.png)

Nato se mu prikaže prijavno okno:

![](../media/login_2.png)
Oznake:
1. Vnos uporabniškega imena
2. Vnos uporabnikovega gesla
3. Gumb za prikaz vnesenega gesla
4. Povezava za ponastavitev gesla

Dvofaktoralna avtentikacija:

![](../media/login_5.png)

Vnosno polje za 2FA geslo:

![](../media/login_6.png)

Ponastavitev gesla:
![](../media/login_3.png)

Elektronsko sporočilo z povezavo za ponastavitev:

![](../media/login_4.png)

Sporočilo po vnesenem elektronskem naslovu:

![](../media/login_7.png)

Obrazec za menjavo gesla (aplikacija nas opozori, če je geslo prešibko)

![](../media/login_8.png)

Password meter se obarva zeleno, če je geslo dovolj varno:

![](../media/login_9.png)

## Opis implementacije

Aplikacija uporablja lastno implementacijo protokola OpenId Connect 1.0 [[1]](https://openid.net/connect/) za pridobivanje poverilnic na čelni strani. Ta protokol temelji na drugem protokolu OAuth 2.0 [[2]](https://oauth.net/2/)

### Hramba poverilnic

Uporabniške poverilnice so sestavljene iz **uporabniškega imena** in **gesla**. Geslo se hrani v zgoščeni obliki s soljo. Zgoščena vrednost se generira z uporabo algoritma BCrypt [[3]](https://en.wikipedia.org/wiki/Bcrypt). S temi poverilnicami BE ugotavlja istovetnost uporabnika. Te poverilnice se nato izmenjajo za žetone v zapisu JWT.

### Podprti tokovi (ang. flows) avtentikacije

Protokol OIDC predvideva več različnih načinov pridobivanja poverilnic. Aplikacija ne podpira polnega nabora teh tokov, ampak le naslednje:

#### Tok z direktnim dostopom (ang. direct access flow, tudi resource owner password flow)

V tem načinu podamo uporabniške poverilnice neposredno na dostopno točko za pridobivanje žetonov.

```curl
curl --location --request POST '${APP_URL}/protocol/oidc/token' \
    --header 'Content-Type: application/x-www-form-urlencoded' \
    --header 'Accept: application/json' \
    --data-urlencode 'grant_type=password' \
    --data-urlencode 'username=${USERNAME}' \
    --data-urlencode 'password=${PASSWORD}'
```

Ta način ni namenjen običajni uporabi, saj se z njim pošiljajo uporabniške poverilnice in je tako smatran za ranljivega. Implementacija tega toka je namenjena izključno testiranju.

#### Tok z obnovitvenim žetonom (ang. refresh token flow)

V tem načinu podamo obnovitveni žeton neposredno na dostopno točko za pridobivanje žetonov.

```curl
curl --location --request POST '${APP_URL}/protocol/oidc/token' \
    --header 'Content-Type: application/x-www-form-urlencoded' \
    --header 'Accept: application/json' \
    --data-urlencode 'grant_type=refresh_token' \
    --data-urlencode 'refresh_token=${REFRESH_TOKEN}'
```

Ta tok je namenjen osveževanju dostopnega žetona (ang. access token), pri daljšem času delovanja aplikacije. Namenjen je izključno temu, da aplikacija lahko sama osvežuje prijavo in drži odprto sejo v uporabnikovem imenu.

#### Tok z avtorizacijsko kodo in dokazilom izmenjave skrivnosti (ang. authorization code flow with PKCE)

Po OIDC specifikaciji priporočen tok za pridobivanje žetonov v zaupanja nevrednih aplikacijah (npr. spletne aplikacije).

Terminologija:
* Odjemalec (krajše OD): spletna aplikacija
* Ponudnik identitete (krajše PI): strežnik, ki izvaja avtentikacijo in avtorizacijo

Tok za razliko od ostalih potrebuje več korakov za pridobitev poverilnic:
1. OD zgenerira 80 znakov dolg niz, z visoko entropijo.
2. OD izračuna izziv (ang. code challenge) na podlagi tega naključnega niza.
3. Naključni niz shrani v brskalnikovo hrambo (Local Storage API).
4. OD preusmeri uporabnika na avtorizacijski vir na PI (`${APP_URL}/protocol/oidc/auth`), pri čemer poda izziv kot del URL - t.i. query parameters.
5. PI preveri če seja za tega uporabnika na tem ip naslovu že obstaja, in če obstaja, preusmeri uporabnika nazaj na OD, pri čemer poda avtorizacijsko kodo kot del URL.
6. Če seja še ne obstaja, bo PI pozval uporabnika naj vnese svoje poverilnice (uporabniško ime in geslo). Če so poverilnice veljavne, bo PI preveril če ima uporabnik vklopljeno dvofaktoralno avtentikacijo. Če je nima, preusmeri uporabnika nazaj na OD, kot pri koraku 5. Če ima 2FA vklopljeno, mu na elektronski naslov pošlje kodo, ki jo mora uporabnik vnesti v PI. Če je koda veljavna, se uporabnika preusmeri nazaj na OD, kot pri koraku 5.
7. OD iz URL-ja prebere avtorizacijsko kodo in jo, skupaj z prej shranjenim naključnim nizom (preveritelj, oz v ang. code verifier) izmenja za žetone z naslednjim zahtevkom:
```curl
curl --location --request POST '${APP_URL}/protocol/oidc/token' \
    --header 'Content-Type: application/x-www-form-urlencoded' \
    --header 'Accept: application/json' \
    --data-urlencode 'grant_type=authorization_code' \
    --data-urlencode 'code=${AUTHORIZATION_CODE}' \
    --data-urlencode 'code_verifier=${CODE_VERIFIER}'
```
8. PI preveri da je avtorizacijska koda ustrezna in da se preveritelj ujema s prej poslanim izzivom. Če so podani argumenti ustrezni, bo ta vir vrnil žetone.

#### Odgovor tokovov

Žetone se pridobi kot odgovor zahtevka na dostopno točko za pridobivanje žetonov. Telo odgovora izgleda tako:

```json
{
    "access_token": "eyJraWQiOiJjYTU5MDU5NC1jODllLTQxN2QtYmZlZS0zNDBhZDliZWQyZTkiLCJhbGciOiJSUzI1NiJ9.eyJzdWIiOiI2NGQ5N2UzYi03ZDJiLTQ5ZGEtYThiNy1iM2YxZjUwMjU3OTgiLCJyb2xlcyI6W10sImlzcyI6Imh0dHA6XC9cL2xvY2FsaG9zdDo4MDgwIiwibmFtZSI6IlRlc3RrbyBUZXN0a292aWMiLCJ0eXAiOiJCZWFyZXIiLCJnaXZlbl9uYW1lIjoiVGVzdGtvIiwiZXhwIjoxNjQ2NjUwOTU2LCJpYXQiOjE2NDY2NTA2NTYsImZhbWlseV9uYW1lIjoiVGVzdGtvdmljIiwiZW1haWwiOiJ0ZXN0QG1haWwuY29tIn0.T5FnjmaaqXxLaIftWco3M8RS8hUlW_0olTlO0oyPBNQ5fbCZtT5fTaxyWDMpYIIuQgESorhaqvUJJjwi-1YNkb0JNXj1_rnjSSL16KVEHY22C7emyH1_1zg4k-fvqP-cZaq3bSWkBINaxsnxPq3_Dq0_bjrTNOW8g0Q_bwD-kuZSQCdJnvFTb4kadI8Hy2U0Rb-hr2FkgSUi6RpBz5-9ejjC5_bIX15HJBgGusfNt5INimNhneg5kuIAQvTvu6T81ADhu9QEFO8jlRoNqX1KMs9Jw6zAFwPfniGC5bNVytd298Aw03lTfEVd7azdCC61fQkABlNKYOBOx4lHksA7gg",
    "expires_in": 300,
    "refresh_token": "eyJraWQiOiJjYTU5MDU5NC1jODllLTQxN2QtYmZlZS0zNDBhZDliZWQyZTkiLCJhbGciOiJSUzI1NiJ9.eyJzdWIiOiI2NGQ5N2UzYi03ZDJiLTQ5ZGEtYThiNy1iM2YxZjUwMjU3OTgiLCJyb2xlcyI6W10sImlzcyI6Imh0dHA6XC9cL2xvY2FsaG9zdDo4MDgwIiwibmFtZSI6IlRlc3RrbyBUZXN0a292aWMiLCJ0eXAiOiJSZWZyZXNoIiwiZ2l2ZW5fbmFtZSI6IlRlc3RrbyIsImV4cCI6MTY0NjY1NDI1NiwiaWF0IjoxNjQ2NjUwNjU2LCJmYW1pbHlfbmFtZSI6IlRlc3Rrb3ZpYyIsImVtYWlsIjoidGVzdEBtYWlsLmNvbSJ9.BwhJujyCHoaAUi8dSIF3RUd5dsnZSprRNcL6U800D6pYMkYnUP8G59aBGiDszzG1eJWPLNrQjJzNOS6PjgvM9_x0ejeIbMK4-ye6_iFUyWmEvBdqjq0dTYJyMrdRbkS5ccwn7C0C3K49v-60b1kp7d0n-JTjg7_9YqjI2L4tSV0YvbIXxRGUuCresuOOeGGNc4ObBPRVyP2qQGf_tt4z3aedP3kcMfrKV8TqCrhPj4_Yi2TT5h-9vwZefxQQsA38KVgOV8OVSxn8LNbQs_zqK0cxWPV2she4L1MT3mopQWoBe2y8bZYLZDsN_jPvqPGqW-aN_SE38sUZCzA0LxsyyA",
    "id_token": "eyJraWQiOiJjYTU5MDU5NC1jODllLTQxN2QtYmZlZS0zNDBhZDliZWQyZTkiLCJhbGciOiJSUzI1NiJ9.eyJzdWIiOiI2NGQ5N2UzYi03ZDJiLTQ5ZGEtYThiNy1iM2YxZjUwMjU3OTgiLCJyb2xlcyI6W10sImlzcyI6Imh0dHA6XC9cL2xvY2FsaG9zdDo4MDgwIiwibmFtZSI6IlRlc3RrbyBUZXN0a292aWMiLCJ0eXAiOiJJZCIsImdpdmVuX25hbWUiOiJUZXN0a28iLCJleHAiOjE2NDY2NTA3MTYsImlhdCI6MTY0NjY1MDY1NiwiZmFtaWx5X25hbWUiOiJUZXN0a292aWMiLCJlbWFpbCI6InRlc3RAbWFpbC5jb20ifQ.ak2wvbHgUhQ8_cXgsNVkVJ_7NGVL27B7k_ig1hMw_drncEbrTY65eg1fY-hNynuIy8pQP5_zrhMms-XC-QbOgiHswAvJAH3wnWpslyXcglQGGIUHJoQXwtkTG1JzsUbTGGRLSj2MWWR9V-KzHbCNQNbbcFbdHg6tpxL4XtAxmwXvYyjaGrRRkOEkwvjHwOkm5o-HmEwGyg775xoMC5RzeO-VX6aCjSK0vb2v29tG-I1cWZ8r2fveT_rdcb6_ceJdd0kpDHZzx9xnTMo_TOh_VCnfn8aD7wb8Om7WfhxtlVAk9KDuX-xcSNGMOLDneqLXA2O3tvi2j76aRp_tlhD5ww"
}
```
### Dodatni tokovi

#### Dvofaktoralna avtentikacija

Dvofaktoralna avtentikacija uporablja elektronsko pošto za dostavljanje kode za preverjanje.

Postopek:
1. Po uspešni prijavi se zgenerira šestmestna številčna koda. Na podlagi identifikatorja seje in te kode, se izračuna zgoščena vrednost (algoritem SHA256). Ta zgoščena vrednost se uporablja za preverjanje da ne napadalec prestreže te kode in jo uporabi namesto uporabnika. 
2. V elektronsko sporočilo prispe koda ter URL, ki omogoča samodejni vnos kode, na podlagi prej izračunane zgoščene vrednosti.
3. Ob vnosu kode, oziroma odprtjem povezave v elektronskem sporočilu, se ponovno izračuna zgoščena vrednost in se tako preveri da ima vnovčitelj kode isto sejo, kot tisti ki jo je zahteval.
4. Če se koda in zgoščena vrednost ujemajo, je uporabnik uspešno prijavljen.

#### Pozabljeno geslo

Uporabnik lahko na prijavnem zaslonu izbere možnost `Forgot password?`, katera mu omogoča ponastavljanje pozabljenega gesla.

Postopek: 
1. Izbrana možnost pozove uporabnika k vnosu elektronskega naslova.
2. Če je elektronski naslov veljaven (uporabniku ta informacija ni razkrita), se bo na ta naslov poslalo sporočilo v katerem bo URL povezava za ponastavitev gesla. Ta povezava bo vsebovala zgoščeno vrednost (algoritem SHA256) identifikatorja uporabnika in ip naslova.
3. Ob odprtju povezave, se bo najprej ponovno izračunala zgoščena vrednost in se preverila da se ujema s prej shranjeno vrednostjo.
4. Če se vrednosti ujemajo, je uporabnik pozvan da vnese novo geslo.
5. Če je geslo pravilno (ustreza politiki gesla), se le-to nastavi za tega uporabnika.

### Podpisovanje žetonov

Vsi izdani žetoni, so zapisani v formatu JWT, natančneje v formatu JWS (Json Web Signature) [[4]](https://datatracker.ietf.org/doc/html/rfc7515). 
Za podpis se uporablja algoritem RS256, ki je sestavljen iz RSA z 2048-bitnim ključem in zgoščene vrednosti SHA256.

Ob zagonu strežnika, se ustvari nov par ključev (če ta še ne obstaja) in se shrani v podatkovno bazo.
