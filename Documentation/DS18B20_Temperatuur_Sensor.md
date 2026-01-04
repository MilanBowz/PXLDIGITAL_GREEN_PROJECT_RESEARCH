# DS18B20 Temperatuur Sensor Documentatie (Bowen)
Categorie: Hardware/Sensor Module

Beschrijving:  
De DS18B20 is een digitale temperatuur sensor die communiceert via het 1-Wire protocol. Deze sensor wordt gebruikt in het LoRa tracking systeem om de omgevingstemperatuur te meten en mee te sturen met de GPS data.

## Properties
- **Type**: Digitale temperatuur sensor
- **Protocol**: 1-Wire (OneWire)
- **Bereik**: -55°C tot +125°C
- **Nauwkeurigheid**: ±0.5°C (tussen -10°C en +85°C)
- **Resolutie**: 9-12 bits configureerbaar (standaard 12-bit)
- **Voeding**: 3.0V tot 5.5V
- **Stroomverbruik**: 1mA actief, 750nA standby
- **Waterdichtheid**: Beschikbaar als ingegoten (IP68) roestvrijstalen probe

### Pinout
| DS18B20 Pin | ESP32 Pin | Functie |
|-------------|-----------|---------|
| DQ (Data) | GPIO 23 | 1-Wire Data lijn |
| VCC | 3.3V | Voeding |
| GND | GND | Ground |

### Verbindingsschema
```
ESP32                DS18B20
GPIO 23 -----------> DQ (Data)
3.3V    -----------> VCC
GND     -----------> GND

Note: Gebruik een 4.7kΩ pull-up resistor tussen DQ en VCC
```

## Software Configuratie

### Benodigde Libraries
```cpp
#include <OneWire.h>            // 1-Wire protocol library
#include <DallasTemperature.h>  // DS18B20 sensor library
```

### Installatie Libraries
Via Arduino IDE Library Manager:
1. **OneWire** by Paul Stoffregen
2. **DallasTemperature** by Miles Burton

### Initialisatie Code
```cpp
// GPIO waar de DS18B20 op aangesloten is
const int oneWireBus = 23;

// Setup OneWire instance
OneWire oneWire(oneWireBus);

// Pass OneWire reference naar Dallas Temperature sensor
DallasTemperature sensors(&oneWire);
```

### Setup
```cpp
void setup() {
    // Start de DS18B20 sensor
    sensors.begin();
    // Geen UART nodig - gebruikt 1-Wire protocol
}
```

## Gebruik

### Temperatuur Uitlezen
```cpp
// Vraag temperatuur metingen aan
sensors.requestTemperatures();

// Lees temperatuur in Celsius (eerste sensor op bus)
float temperatureC = sensors.getTempCByIndex(0);

// Lees temperatuur in Fahrenheit (optioneel)
float temperatureF = sensors.getTempFByIndex(0);
```

### Print Functie
De code bevat een helper functie om temperaturen te printen:

```cpp
void printGetTemps() {
    sensors.requestTemperatures(); 
    float temperatureC = sensors.getTempCByIndex(0);
    float temperatureF = sensors.getTempFByIndex(0);
    
    Serial.print(temperatureC);
    Serial.println("ºC");
    Serial.print(temperatureF);
    Serial.println("ºF");  
}
```

### Integratie in LoRa Bericht
De temperatuur wordt meegestuurd in het JSON bericht:

```cpp
sensors.requestTemperatures();
float temperatureC = sensors.getTempCByIndex(0);

// Voeg temperatuur toe aan JSON payload
snprintf(buf, sizeof(buf), 
    "{\"device\":\"ESP32-GPS\",\"lat\":%.6f,\"lon\":%.6f,\"temp\":%.2f}", 
    lat, lon, temperatureC);
```

## Technische Details

### 1-Wire Protocol
- **Voordelen**: 
  - Slechts één data pin nodig
  - Meerdere sensoren op één bus mogelijk
  - Uniek 64-bit ROM ID per sensor
- **Nadelen**:
  - Trager dan I2C/SPI
  - Vereist pull-up resistor

### Resolutie en Meettijd
| Resolutie | Meettijd | Precisie |
|-----------|----------|----------|
| 9-bit | 93.75 ms | 0.5°C |
| 10-bit | 187.5 ms | 0.25°C |
| 11-bit | 375 ms | 0.125°C |
| 12-bit (default) | 750 ms | 0.0625°C |

### Multiple Sensoren
De DS18B20 ondersteunt meerdere sensoren op één bus:

```cpp
// Aantal sensoren tellen
int numberOfDevices = sensors.getDeviceCount();

// Loop door alle sensoren
for(int i = 0; i < numberOfDevices; i++) {
    float temp = sensors.getTempCByIndex(i);
    Serial.print("Sensor ");
    Serial.print(i);
    Serial.print(": ");
    Serial.println(temp);
}
```

## Foutafhandeling

### Error Codes
De sensor retourneert `-127.00°C` bij een fout:

```cpp
float temperatureC = sensors.getTempCByIndex(0);

if (temperatureC == -127.00) {
    Serial.println("Fout: Sensor niet gevonden of niet aangesloten!");
} else {
    Serial.print("Temperatuur: ");
    Serial.println(temperatureC);
}
```

### Veelvoorkomende Fouten
| Probleem | Mogelijke Oorzaak | Oplossing |
|----------|-------------------|-----------|
| -127.00°C | Sensor niet verbonden | Controleer bedrading |
| | Pull-up resistor ontbreekt | Plaats 4.7kΩ resistor |
| | Verkeerde pin | Controleer GPIO 23 |
| 85.00°C (altijd) | Sensor niet geïnitialiseerd | Roep `sensors.begin()` aan |

## Optimalisatie

### Snellere Metingen
```cpp
// Verlaag resolutie voor snellere metingen
sensors.setResolution(9); // 9-bit = 93.75ms
```

### Parasitic Power Mode
De DS18B20 kan ook zonder VCC pin werken (parasitic mode):
- Sluit alleen DQ en GND aan
- VCC blijft los
- Gebruik sterkere pull-up (2.2kΩ in plaats van 4.7kΩ)
- **Niet aanbevolen** voor dit project (minder betrouwbaar)

### Energiebesparing
In combinatie met ESP32 sleep mode:

```cpp
// Voor sleep
sensors.requestTemperatures();
float temp = sensors.getTempCByIndex(0);
// Verstuur data
// Ga slapen - sensor gebruikt automatisch minimal power
esp_light_sleep_start();
```

## Project Implementatie

### Wanneer Wordt de Temperatuur Gemeten?
- Elke 5 seconden (samen met GPS data)
- Voor verzending via LoRa
- Bij zowel geldige als ongeldige GPS data

### Data Format
```json
{
  "device": "ESP32-GPS_Maxime",
  "lat": 51.234567,
  "lon": 4.567890,
  "alt": 45.00,
  "sats": 8,
  "ts": 1234567890,
  "temp": 21.50,
  "light": "Het is licht buiten, nachtlamp staat uit"
}
```

## Troubleshooting Checklist

1. ✓ Controleer 3.3V voeding
2. ✓ Controleer GND verbinding
3. ✓ Controleer GPIO 23 (DQ) verbinding
4. ✓ Controleer 4.7kΩ pull-up resistor aanwezig
5. ✓ Verify `sensors.begin()` wordt aangeroepen
6. ✓ Test met simpele temperatuur print code
7. ✓ Controleer library versies (OneWire + DallasTemperature)

## Referenties
- [DS18B20 Datasheet](https://datasheets.maximintegrated.com/en/ds/DS18B20.pdf)
- [Praktische ESP32 + DS18B20 handleiding (Random Nerd Tutorials)](https://randomnerdtutorials.com/esp32-ds18b20-temperature-arduino-ide)
