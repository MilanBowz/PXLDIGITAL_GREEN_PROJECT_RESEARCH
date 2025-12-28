# 🌤️ LDR Lichtmeting voor LoRa-project
(lennert)
 
Dit project gebruikt een **LDR (Light Dependent Resistor)** om omgevingslicht te meten en een LED automatisch aan of uit te schakelen op basis van de lichtintensiteit.  
Het is bedoeld als onderdeel van een groter **LoRa-sensornetwerk**, waar de lichtstatus later via LoRa kan worden verzonden.

---

## ⚙️ Hardwareoverzicht 

**Bestand:** ![`LDRadded.jpg`](../Afbeeldingen/LDRadded.jpg) 

De schakeling bestaat uit:
- Een **LDR** in serie met een **weerstand** (spanningsdeler).
- Een **analoge ingangspin (GPIO34)** die de spanning van de spanningsdeler meet.
- Een **LED op GPIO25** die automatisch aan/uit gaat.

**Schema:**
- LDR verbonden tussen 3.3V en het analoge meetpunt (A0 / GPIO34).  
- Weerstand (~10kΩ) tussen meetpunt en GND.
- LED met serieweerstand (~220Ω) tussen GPIO25 en GND.

---

## 🧠 Functie van de code

De code leest de lichtintensiteit via de LDR en vergelijkt die met een ingestelde **drempelwaarde (threshold)**.  
Afhankelijk van het resultaat schakelt het systeem een LED aan of uit.

```
#include "arduino.h"

const int ldrPin = 34;      //ADC pin voor LDR
const int LED = 25;         //GPIO pin voor LED
const int threshold = 200; //drempelwaarde LDR led aan/uit

void setup() {
    // baudrate van de seriale communicatie
    Serial.begin(9600);
    pinMode(LED, OUTPUT);
}

void loop() {
    // Lees LDR & stuur LED aan
        int ldrValue = analogRead(ldrPin);
        bool zonOp = (ldrValue > threshold);

        if (zonOp){
            digitalWrite(LED, LOW); // Nachtlamp uit bij daglicht
        } else {
            digitalWrite(LED, HIGH); // Nachtlamp aan bij duisternis
        }
        
        const char* lightStr = zonOp ? "Het is licht buiten, nachtlamp staat uit" : "Het is donker buiten, nachtlamp staat aan";
        char buf[256];
        snprintf(buf, sizeof(buf), "light: %.2f", lightStr);
    }
```
