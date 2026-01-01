# Battery & Power Management Module (Bowen)
Categorie: Power / Hardware Module  

Beschrijving:  
Deze module verzorgt de energievoorziening van het ESP32 LoRa verzendsysteem en zorgt 
voor autonome werking, veilig opladen en betrouwbaar stroombeheer.

---

## Properties
- **Batterij**: Lithium-ion 103450, 3,7 V, 2000 mAh  
- **Laadmodule**: TP4056 USB Type-C
- **DC-DC Converter**: XL6019 Step-Up (3,3 V–35 V → 5 V/6 V/9 V/12 V/24 V)  
- **Aan/uit-schakelaar**: Mechanisch, in serie met de batterij voor volledige uitschakeling  
- **Stroomverbruik ESP32**: ~90 mA (light-sleep), ~150 mA (actief met alle sensoren en modules)

---

## Hardware Aansluiting
Het schema hieronder toont hoe de batterij is verbonden met de TP4056 laadmodule, DC-DC step-up converter,  
ESP32 en sensoren. De aan/uit-schakelaar is geplaatst tussen de batterij en TP4056 om de voeding te kunnen onderbreken.

![Batterij- en voedingsschema](../Afbeeldingen/Sender_Connections.jpg)

**Uitleg voeding:**
- **Batterij (3,7 V)**: levert stroom aan de TP4056 laadmodule.  
- **TP4056 laadmodule**: regelt veilig opladen via USB-C, beschermt tegen overladen en diepontladen. De batterij wordt via B+ en B- aangesloten.  
- **DC-DC step-up converter**: de 3,7 V van de batterij wordt verhoogd naar 5 V.  
  - **Out+** van de converter gaat naar **Vin** van de ESP32, waardoor de ESP32 wordt gevoed.  
  - **Out-** wordt verbonden met **GND** van de ESP32.  
- **Aan/uit-schakelaar**: geplaatst tussen batterij en TP4056, hiermee kan het hele systeem mechanisch worden uitgeschakeld.

---

### Testen en Metingen
De stroommetingen zijn uitgevoerd met een USB digital tester aangesloten op de USB-C connector van de ESP32.  
Hierdoor konden we het totale stroomverbruik van de ESP32 met alle sensoren meten.

- **Light-sleep**: ~90 mA  
  - ESP32 en sensoren in low-power modus, minimale achtergrondprocessen actief  
- **Actief**: ~150 mA  
  - ESP32 actief met alle sensoren en modules (GPS, LoRa, temperatuursensor)

### Light-Sleep Implementatie
```cpp
#include "esp_sleep.h"  // ESP32 sleep modes header

// Light sleep tussen verzendintervallen
esp_sleep_enable_timer_wakeup(SEND_INTERVAL_MS * 1000); // timer wake-up (parameter in microseconden)
Serial.println("Entering light sleep...");
esp_light_sleep_start(); // start light sleep
```
**Uitleg:**  
- De ESP32 gaat in light-sleep voor de ingestelde interval (`SEND_INTERVAL_MS`) tussen LoRa-verzendingen.  
- Hierdoor wordt het stroomverbruik verminderd (~90 mA), terwijl het systeem automatisch wakker wordt om nieuwe data te verzenden.
---

[Bekijk de video voor meting](../Afbeeldingen/MetingenVid.mp4)

## Problemen en Oplossingen

### 1. Hoog stroomverbruik
**Probleem**: Continu actieve ESP32 en sensoren verbruikten onnodig stroom.  
**Oplossing**: ESP32 schakelt tussen active en deep-sleep modus.

### 2. Geen Aan/Uit-knop
**Probleem**: Systeem bleef permanent onder spanning, geen controle tijdens transport.  
**Oplossing**: Mechanische schakelaar toegevoegd tussen batterij en TP4056.  

### 3. Geen inzicht in verbruik
**Probleem**: Onbekend stroomverbruik.  
**Oplossing**: USB digital tester gebruikt voor realtime metingen.  

