# Application Note – ESP32 LoRa Zender (Bowen)
**Titel**: ESP32 LoRa zender met GPS en temperatuurmetingen  
**Auteur**: Bowen  

## Abstract
Deze application note beschrijft een zelfgebouwde ESP32-LoRa-zender die GPS- en temperatuursdata bundelt in een JSON-payload en periodiek uitstuurt. Doel: een eenvoudig, batterijgevoed prototype dat binnen ETSI-regels (868 MHz) betrouwbaar communiceert. De opzet gebruikt LoRa E220, Neo-6M/M8N GPS, DS18B20 en een Li-ion/TP4056/DC-DC voedingsketen. Resultaten: ~1,1 km bereik in bebouwde omgeving, stabiele GPS-fix outdoor (3–5 m), consistente temperatuurmetingen met correcte pull-up, stroomverbruik ~90 mA (light-sleep) en ~150 mA (actief). Conclusie: het project is stabiel genoeg, maar kan nog worden verbeterd met diepere slaap en betere indoor GPS.

## 1. Introductie
Doel: een compacte, autonome zender die locatie- en temperatuursdata via LoRa verstuurt binnen ETSI 868 MHz. Probleemstelling: langeafstandsmeting met laag vermogen en beperkte bandbreedte, maar toch betrouwbare sensor (GPS + temperatuur). Onderzoeksvraag: hoe bouw je als student een ESP32-configuratie die legaal en stabiel data verstuurt met voldoende bereik en aanvaardbaar verbruik.

## 2. Materiaal en methode
### 2.1 Hardware
- ESP32 Dev Board (centrale MCU)  
- LoRa E220 (868/915 MHz, 22 dBm) + 868 MHz omni-antenne  
- Neo-6M / NEO-M8N GPS-module  
- DS18B20 (waterdichte probe, 4.7 kΩ pull-up)  
- Li-ion 103450 (3.7 V, 2000 mAh) + TP4056 lader + DC-DC (step-up naar 5 V)  
- Breadboard/soldeerplaat, jumper wires, plastic behuizing

### 2.2 Software en libraries
- IDE: VS Code + PlatformIO, Arduino IDE  
- Libraries: `LoRa_E220`, `TinyGPSPlus`, `DallasTemperature`, `esp_sleep.h`  
- Tools: Python, Node-RED

### 2.3 Schema en integratie
- Bedradingsschema: zie [Afbeeldingen/Sender_Connections.jpg](../Afbeeldingen/Sender_Connections.jpg).  
- Kernverbindingen: E220 op GPIO16/17 (UART2), GPS op GPIO4/5 (UART1), DS18B20 op GPIO23 met 4.7 kΩ pull-up, voeding via Li-ion → TP4056 → DC-DC 5 V → ESP32 Vin.  
- Flow: sensoren lezen → JSON-payload bouwen → LoRa E220 zenden → light-sleep tot volgende interval.

### 2.4 Methode
- LoRa-config binnen ETSI (14 dBm, 1% duty cycle, 868 MHz kanaal).  
- GPS: periodiek uitlezen, data alleen gebruiken bij geldige fix.  
- DS18B20: OneWire met verplichte pull-up, temperatuur toevoegen aan payload.  
- Power: light-sleep tussen verzendintervallen, verbruik meten met USB digital tester.

## 3. Resultaten
### 3.1 LoRa-transmissie
- Opstelling: ESP32 ↔ LoRa E220 via UART2, eerst breadboard, daarna soldeerplaat.  
- Bereik: ~1,1 km betrouwbaar in bebouwde omgeving met 868 MHz omni-antenne.  
- Compliance: 14 dBm EIRP, 1% duty cycle, kanaal binnen 868,0–868,6 MHz.  
- Receiver: Raspberry Pi 5 had initieel UART-issues, na juiste configuratie gefixt en nu stabiele ontvanger. Tweede ESP32 werd tussentijds gebruikt tijdens debug. (Zender gebouwd en Pi-config mee opgelost.)

### 3.2 GPS
- Fix: indoor geen fix, outdoor fix binnen minuten, nauwkeurigheid ~3–5 m.  
- Behuizing: cold start trager in kunststof behuizing, stabiel na eerste fix.  
- Integratie: lat/lon/alt/sats toegevoegd aan payload.

### 3.3 Temperatuur (DS18B20)
- Zonder 4.7 kΩ pull-up: instabiel/onrealistisch.  
- Met pull-up: stabiel en realistisch, waarden in payload.  
- Waterdichte probe bruikbaar voor buitenmetingen.

### 3.4 Power en slaap
- Verbruik: ~90 mA (light-sleep), ~150 mA (actief met LoRa+GPS+temp).  
- Voeding: Li-ion + TP4056 + DC-DC levert stabiele 5 V, geen resets tijdens TX/GPS.  
- Sleep: timer-gebaseerde light-sleep tussen zenden.

### 3.5 Mechanische/electrische afwerking
- Layout: GPS boven, antenne voor, USB-C achter, DS18B20 naar buiten.  
- Soldeerplaat voor betrouwbaarheid, korte sporen en stabiele connectoren.  
- Multimetercheck op kortsluiting/continuïteit, daarna validatie onder spanning.

## 4. User Experience
- Gebruik: aanzetten, buiten wachten op GPS-fix, data wordt automatisch verzonden.  
- Feedback: seriële logging tijdens ontwikkeling, LED-indicatie mogelijk in productie.  
- Mogelijke verwarring: indoor geen GPS-fix.

## 5. Discussie
- ETSI-compliance: In de eerste fase werd niet aan de 14 dBm EIRP-limiet gehouden (te hoog zendvermogen ingesteld). Na review van ETSI EN 300 220 regelgeving werd dit gecorrigeerd naar 14 dBm, 1% duty cycle en kanaal 868,0–868,6 MHz. Dit was een belangrijke leerpunt voor regelgeving.
- LoRa-bereik voldoet voor bebouwde omgeving met correcte instellingen; verdere winst via antennehoogte/plaatsing.
- Indoor GPS blijft beperkt; externe antenne of assisted GPS kan helpen.  
- Stroomverbruik kan omlaag met deep-sleep, lagere zendsnelheid of langere intervallen.  
- Pi5-ontvanger: initieel UART-problemen, nadien opgelost door juiste configuratie, ESP32-ontvanger fungeerde als fallback tijdens debug.

## 6. Conclusie
De ESP32-LoRa zender werkt stabiel binnen ETSI-limieten en haalt ~1,1 km bereik met GPS- en temperatuurlogging. De voeding (Li-ion/TP4056/DC-DC) is robuust, verbruik is acceptabel maar verder te optimaliseren. GPS is betrouwbaar outdoor, indoor blijft een beperking. Volgende stappen: (1) deep-sleep toepassen, (2) batterijstatus zichtbaar maken, (3) eigen PCB ontwerpen voor compacte, robuuste opbouw, (4) behuizing verder afdichten.
