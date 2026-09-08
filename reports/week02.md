1. Johdanto
SNMP (Simple Network Management Protocol) on protokolla, jonka avulla voidaan seurata ja hallita verkon laitteita. Sen avulla voidaan hakea laitteelta erilaisia tietoja, kuten laitteen nimi, käyttöjärjestelmä ja verkkorajapintojen tila.

SNMPä käytetään verkon laitteiden valvontaan. Sen avulla voidaan esimerkiksi tarkistaa, ovatko laitteet ja niiden verkkorajapinnat toiminnassa sekä kerätä tietoa laitteiden tilasta. Tietoja voidaan kerätä keskitetysti ilman, että jokaiseen laitteeseen tarvitsee kirjautua erikseen.

2. Asennus

SNMP-agentti asennettiin web1-, db1- ja branch-client-kontteihin. Ensin päivitettiin pakettien tiedot komennolla apt update ja tämän jälkeen asennettiin SNMP-agentti komennolla:
apt install snmp snmpd -y

Asetustiedostoa muokatessa huomattiin, että nano ei ollut kaikissa konteissa valmiiksi asennettuna. Se asennettiin komennolla:
apt install nano -y

SNMP-agentin toiminta tarkistettiin ja tarvittaessa palvelu käynnistettiin komennolla service snmpd start.

SNMP-agentin asetuksia muutettiin tiedostossa /etc/snmp/snmpd.conf. Community stringiksi asetettiin public:
rocommunity public

Lisäksi SNMP-palvelu asetettiin kuuntelemaan UDP-porttia 161:
agentaddress udp:161

Muutosten jälkeen SNMP-palvelu käynnistettiin uudelleen komennolla:
service snmpd restart

Myöhemmin SNMP-kyselyissä esimerkiksi system, sysName.0 ja ifDescr eivät toimineet nimillä, koska MIB-tietoja ei ollut saatavilla. Tämän takia kyselyissä käytettiin vastaavia numeerisia OID-tunnuksia.

3. Kerätyt tiedot

Suoritin ansible-kontista SNMP-kyselyn web1-laitteelle:
snmpwalk -v2c -c public web1 1.3.6.1.2.1.1

Kysely palautti tietoa web1-laitteen järjestelmästä, kuten laitteen nimen, käyttöjärjestelmän ja uptime-tiedon.

3.1 System info

SNMP-kyselyillä tarkistettiin laitteen nimi, käyttöjärjestelmän tiedot ja uptime.

snmpget -v2c -c public web1 1.3.6.1.2.1.1.5.0
iso.3.6.1.2.1.1.5.0 = STRING: "web1"

snmpget -v2c -c public web1 1.3.6.1.2.1.1.1.0
iso.3.6.1.2.1.1.1.0 = STRING: "Linux web1 6.18.33.2-microsoft-standard-WSL2 #1 SMP PREEMPT_DYNAMIC Thu Jun 18 21:54:43 UTC 2026 x86_64"

snmpget -v2c -c public web1 1.3.6.1.2.1.1.3.0
iso.3.6.1.2.1.1.3.0 = Timeticks: (356794) 0:59:27.94

Tulosten perusteella laitteen nimi on web1, käyttöjärjestelmä on Linux ja uptime kyselyhetkellä oli noin 59 minuuttia 28 sekuntia.

4. Verkkorajapinnat

Verkkorajapinnat haettiin komennolla:
snmpwalk -v2c -c public web1 1.3.6.1.2.1.2.2.1.2

iso.3.6.1.2.1.2.2.1.2.1 = STRING: "lo"
iso.3.6.1.2.1.2.2.1.2.2 = STRING: "eth0"
iso.3.6.1.2.1.2.2.1.2.4389 = STRING: "eth1"

Löytyi siis kolme verkkorajapintaa: `lo`, `eth0` ja `eth1`.

Rajapintojen tila tarkistettiin komennolla:
snmpwalk -v2c -c public web1 1.3.6.1.2.1.2.2.1.8
iso.3.6.1.2.1.2.2.1.8.1 = INTEGER: 1
iso.3.6.1.2.1.2.2.1.8.2 = INTEGER: 1
iso.3.6.1.2.1.2.2.1.8.4389 = INTEGER: 1

Kaikki kolme rajapintaa saivat arvon 1, joka tarkoittaa, että rajapinnat ovat toiminnassa. lo on loopback-rajapinta ja eth0 sekä eth1 ovat verkkorajapintoja.
Selvitettiin vielä mihin verkkorajapintaan laitteen IP-osoitteet kuuluvat.
snmpwalk -v2c -c public web1 1.3.6.1.2.1.4.20.1.2

iso.3.6.1.2.1.4.20.1.2.10.10.20.101 = INTEGER: 4389
iso.3.6.1.2.1.4.20.1.2.127.0.0.1 = INTEGER: 1
iso.3.6.1.2.1.4.20.1.2.172.20.20.9 = INTEGER: 2

Eli varsinainen web1:n verkko-osoite 10.10.20.101 on rajapinnassa eth1.

5. OID-analyysi

OID	Tarkoitus
sysName.0	= laitteen nimi.
sysDescr.0	= laitteen järjestelmän ja käyttöjärjestelmän kuvaus.
sysUpTime.0	= kuinka kauan laite tai SNMP-agentti on ollut käynnissä.
ifDescr	    = verkkorajapintojen nimet
ifOperStatus	= verkkorajapintojen toimintatila, esimerkiksi UP tai DOWN.

| Laite         | Nimi          | Käyttöjärjestelmä | Uptime      |
| ------------- | ------------- | ----------------- | ----------- |
| web1          | web1          | Linux             | 59 min 28 s |
| db1           | db1           | Linux             | 2 min 8 s   |
| branch-client | branch-client | Linux             | 35,64 s     |

6. Pohdinta

SNMP:n avulla verkon laitteiden tilaa ja tietoja voidaan seurata helpommin. Sen avulla voidaan esimerkiksi tarkistaa laitteiden toimintaa ja verkkojen tilaa ilman, että jokaiseen laitteeseen tarvitsee kirjautua erikseen.

SNMP:llä voidaan kerätä esimerkiksi laitteen nimeä, järjestelmän tietoja, uptimea ja verkkojen tietoja. 

SNMPv2c community string ei ole salattu. Sen takia se ei ole kovin turvallinen tapa suojata SNMP-yhteyttä, koska stringi voidaan saada selville verkkoa kuuntelemalla.

SNMPv3 kannattaa käyttää silloin, kun tarvitaan enemmän tietoturvaa. Siinä on parempi tunnistautuminen ja viestit voidaan salata. Siksi se sopii paremmin esimerkiksi tuotantoverkkoihin.

Opin tehtävän aikana, miten SNMP toimii ja miten sen avulla voidaan hakea tietoja verkon laitteista. Opin myös asentamaan ja konfiguroimaan SNMP-agentin sekä tekemään SNMP-kyselyitä toisesta kontista.

Lisäksi opin käyttämään OID-tunnuksia tietojen hakemiseen. Tehtävän aikana tuli myös vastaan pieniä ongelmia, kuten MIB-nimien toimimattomuus ja SNMP-agentin kuunteluasetukset. Niiden selvittäminen auttoi ymmärtämään paremmin, miten SNMP käytännössä toimii.


7.Tekoälyn käyttö

Hyödynsin tehtävän aikana tekoälyä apuna SNMP komentojen ja OID-tunnusten ymmärtämisessä sekä ongelmatilanteiden selvittämisessä. Tekoäly auttoi myös raportin rakenteen ja tekstin muotoilussa. Tehtävän komennot suoritettiin itse ja tarkistin saadut tulokset ympäristössäni.
