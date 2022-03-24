# Dodajanje uporabnikov

Uporabnike lahko dodajajo samo uporabniki s skrbniškimi pravicami.

Uporabnikom, ki imajo ustrezne pravice, se v naslovni vrstici, poleg uporabniške slike, prikaže še ikona za skrbniško nadzorno ploščo:

![](../media/users_1.png)

*(barva je lahko drugačna)*

Nato v nadzorni plošči izberejo `New user`:

![2022-03-24](https://user-images.githubusercontent.com/24944462/159912339-54992597-39d6-4675-8f65-165dda492cd0.png)

Ko uporabniki z ustreznimi pravicami izberejo  `New user`, se prikaže obrazec za dodajanje novega uporabnika. V ta obrazec je obvezno vnesti 
1. Vzdevek
2. Geslo
3. Potrditev gesla (se mora ujemati z geslom)
4. Ime
5. Priimek
6. Elektronski naslov
7. Pravice novega uporabnika

![userform2](https://user-images.githubusercontent.com/24944462/159921550-d03ca318-4fc2-4168-bfee-d1b44d91e381.png)


## Opis implementacije

Čelni del aplikacije je zgrajen na platformi Angular. Vsi obrazci so zgrajeni po istem principu - kot primer je spodaj opisana implementacija obrazca za dodajanje uporabnika. 

###Implementacija obrazca za dodajanje uporabnika

Obrazec za dodajanje uporabnika se nahaja v svoji komponenti. Zraven komponente se v isti mapi nahajajo tudi ročno spisani na dodajanje uporabnika vezani validatorji, ki ne potrebujejo povezave z bazo. Podatki iz obrazca se pošiljajo na bazo s pomočjo uporabniškega servisa, kjer se nahajajo tudi tisti na uporabnika vezani validatorji, ki so povezani z bazo. Za korektno delovanje je potrebno tudi preverjanje tipov podatkov, kar je narejeno z uporabniškimi vmesniki.
