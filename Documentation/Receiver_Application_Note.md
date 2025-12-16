## LoRa-ontvanger: overzicht

Voor dit project heb ik een ontvanger gebouwd op basis van een Raspberry Pi die via UART is verbonden met een E220 LoRa-module. De ontvanger ontvangt JSON-pakketten die door onze ESP32‑zender worden verstuurd en verwerkt deze automatisch. Deze opstelling vormt de kern van mijn LoRa‑ontvanger.

**Belangrijkste doelen van het ontwerp:**

* een stabiele en betrouwbare UART‑verbinding opzetten
* LoRa‑pakketten correct en consistent ontvangen
* JSON‑data foutloos inlezen en valideren
* de ontvangen gegevens automatisch doorgeven aan Node‑RED voor verdere verwerking en visualisatie
---

## Hardware‑architectuur van de ontvanger

De ontvanger bestaat uit een Raspberry Pi waarop de E220 LoRa‑module is aangesloten via de UART‑interface. De Raspberry Pi fungeert als centrale verwerkingsunit en ontvangt de LoRa‑data, die vervolgens softwarematig wordt afgehandeld.

**Belangrijkste hardware‑componenten:**

* Raspberry Pi
* E220 LoRa‑module
* UART‑verbinding (TX/RX)
* LCD‑scherm voor lokale visualisatie

---

## Ontvangst en verwerking van LoRa‑data

De ESP32‑zender bundelt data van meerdere sensoren in één JSON‑pakket en verstuurt dit via LoRa. Aan de ontvangerzijde wordt dit pakket via UART ingelezen.

De ontvangen data wordt:

1. uitgelezen via de seriële poort
2. gecontroleerd op geldige JSON‑structuur
3. omgezet naar bruikbare velden (temperatuur, licht, GPS, status, tijdstip)
4. doorgestuurd naar Node‑RED

Door het gebruik van JSON blijft de datastructuur overzichtelijk en eenvoudig uitbreidbaar.

---

## Node‑RED‑dataverwerking

In Node‑RED heb ik een volledige flow opgebouwd die de ontvangen JSON‑data uitleest, controleert en verwerkt. De flow is verantwoordelijk voor:

* het parsen van JSON‑berichten
* het detecteren van actieve of inactieve devices
* het opslaan en doorsturen van meetwaarden
* het aansturen van het dashboard en het LCD‑scherm

Voor een gedetailleerde beschrijving van deze flow verwijs ik naar:

[Receiver_Node-RedFlow.md](Receiver_Node-RedFlow.md)

---

## LCD‑scherm en lokale visualisatie

Naast de verwerking in Node‑RED heb ik een fysiek LCD‑scherm gekoppeld aan de ontvanger. De volledige aansturing van dit scherm – welke tekst wanneer wordt getoond – is ook in Node‑RED opgebouwd en gebeurt via een LCD‑node.

Op het LCD‑scherm wordt automatisch weergegeven:

* welke devices momenteel verbonden zijn
* de actuele temperatuur per device
* de verbindingsstatus (actief / geen recente data)

Hierdoor is het niet nodig om het dashboard te openen om snel de status van het systeem te controleren.

---

## Dashboard voor de gebruiker

Voor de eindgebruiker is er een Node‑RED dashboard voorzien. Dit dashboard toont de belangrijkste gegevens in real‑time:

* temperatuur
* lichtniveau
* GPS‑gegevens
* tijdstip van de laatste update
* verbindingsstatus per device

Het dashboard biedt een duidelijk en overzichtelijk beeld van alle inkomende LoRa‑data.

---

## Totaalconcept van de ontvanger

Het volledige ontvangerconcept bestaat uit:

* een Raspberry Pi met E220 LoRa‑module
* UART‑communicatie
* ontvangst en verwerking van JSON‑data
* Node‑RED voor dataverwerking
* een real‑time dashboard
* een fysiek LCD‑scherm met statusinformatie

De data van de ESP32‑zender wordt automatisch ontvangen, verwerkt en zowel grafisch als tekstueel gepresenteerd.

---

## Gebruikerservaring

Tijdens de ontwikkeling heb ik sterk rekening gehouden met de eindgebruiker. Het systeem werkt volledig automatisch en vereist geen manuele bediening.

**Ervaring voor de gebruiker:**

* het LCD‑scherm toont onmiddellijk welke devices actief zijn
* de temperatuur en status zijn direct zichtbaar
* het dashboard visualiseert alle metingen live

Op basis van de resultaten kan ik besluiten dat de ontvanger voldoet aan mijn verwachtingen. De LoRa‑ontvangst is stabiel en de verwerking in Node‑RED verloopt zonder noemenswaardige problemen.

---

## Conclusie

Samengevat werkt mijn ontvangerproject betrouwbaar en gebruiksvriendelijk. Dankzij de combinatie van LoRa, Node‑RED en een fysiek LCD‑scherm kan de gebruiker op een duidelijke en intuïtieve manier de status en meetgegevens van alle devices opvolgen. Het systeem voldoet aan de vooropgestelde basisvereisten en is klaar voor verdere uitbreiding in de toekomst.
