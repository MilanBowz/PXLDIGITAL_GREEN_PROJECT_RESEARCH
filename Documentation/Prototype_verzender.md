# Prototype verzender: Solderen, ...

## Verzender prototype 1: ESP32 [Milan]

### Componenten

1. Gaatjesprint 50 * 70 mm
2. Microcontroller: ESP32 
3. Stijve geleidende draad en jumperkabels
4. LoRa communicatie LoRa-module: E220
5. Multimeter om correcte verbinding en kortsluitingen te meten
6. Soldeerstation en bijhorende hulpmiddelen
### Werkwijze: prototype 1 & 2

1. **Solderen van female pinheaders**
	1. Eerst moet worden bepaald waar elk component wordt aangesloten aan de hand van het schema.
	2. De soldering start bij de microcontroller, gevolgd door de LoRa-module.
	3. Elke verbinding gecontroleerd met een multimeter om correcte verbinding en het ontbreken van kortsluitingen te verifiëren.
2. **Bedrading (stijve geleidbare draad)**
	1. Begin met de aansluiting van Vcc & GND
3. **Controle verbinding**
	Na het solderen wordt de verbinding tweemaal gecontroleerd. De controle focust zowel op het vermijden van kortsluitingen als op het garanderen van een betrouwbare elektrische verbinding
	1. ~ 5 minuten na het solderen
	2. ~ 24 uur na het solderen, om mogelijke problemen van warmte of mechanische spanning te detecteren.
4. **Extra isolatie**
	- Ter bescherming tegen waterschade en het risico op kortsluiting wordt een isolerende coating aangebracht.
    - Tijdens dit proces worden gevoelige componenten tijdelijk verwijderd om beschadiging te voorkomen.

## Verzender prototype 2: Wemos D1 mini pro [Milan]

### Werkwijze

De werkwijze voor prototype 2 is identiek aan die van prototype 1, met als voornaamste verschil dat de ESP32 is vervangen door een Wemos D1 mini Pro. Daarnaast wordt gebruikgemaakt van een kleinere gaatjesprint met afmetingen van 40 × 60 mm.

Deze aanpassing heeft als doel te evalueren of het ontwerp kan worden verkleind naar een compacter formaat, zonder negatieve impact op de functionaliteit, betrouwbaarheid of soldeerkwaliteit van het prototype.
### Conclusie

Uit prototype 2 blijkt dat het technisch mogelijk is om het ontwerp verder te verkleinen naar 40 × 60 mm.

Bij dit formaat wordt de integratie van andere componenten, zoals sensoren en batterij veel complexer. De beperkte beschikbare ruimte vereist een zeer doordachte plaatsing en soldering, waarbij manuele assemblage steeds moeilijker wordt.

Om deze reden is dit compact ontwerp beter geschikt voor machinale ontwikkeling, waar hogere precisie kunnen worden gegarandeerd.


![[Afbeeldingen/Solderen_1.png]]
</br>*Solderen prototype 1 & 2: achterkant* </br>
![[Afbeeldingen/Solderen_2.png]]
</br> *Solderen prototype 1 & 2: voorkant* </br>

Om deze 2 prototypes te testen kan FT_FIXED_TRANSMISSION geraadpleegd worden.




