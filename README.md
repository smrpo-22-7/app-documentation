# SCRUM DOKUMENTACIJA

### Kazalo

1. Prijava
    1. [Prijava v sistem](./content/LOGIN.md)
    2. [Uporabniški profil](./content/USER_PROFILE.md)
2. [Dodajanje uporabnikov](./content/USERS.md)
3. [Dodajanje projekta](./content/PROJECTS.md)
4. [Ustvarjanje novega sprinta](./content/SPRINTS.md)

## Terminologija

V tem dokumentu se uporabljajo naslednji izrazi in okrajšave:

* **Zaledni del** (ang. backend): Strežniški del aplikacije, ki skrbi za podatke, avtentikacijo in avtorizacijo.
* **BE**: Okrajšava angleške besede *backend*, pomeni **zaledni del** aplikacije.
* **Čelni del** (ang. frontend): Spletna aplikacija, ki služi kot uporabniški vmesnik za aplikacijo.
* **FE**: Okrajšava angleške besede *frontend*, pomeni **čelni del** aplikacije.
* **DB**, **baza**: Podatkovna baza SQL (PostgreSQL), ki shranjuje podatke. Z njo upravlja zaledni del.
* **Aplikacija**: Skupni izraz za zaledni in čelni del.
* **OIDC**: Okrajšava za protokol *OpenId Connect 1.0*
* **JWT**: Okrajšava za format poverilnic *Json Web Token*
* **Vir**: Dostopna točka na strežniku. Primer: `/v1/projects` je URL enega vira, na katerega se strežnik odziva.
