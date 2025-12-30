
# GPS Module  [Maxime] 

Categorie: Hardware/Sensor Module  
Beschrijving:  
De GPS-module (NEO-8M) geïntegreerd in de TTGO T-Beam zorgt voor positiebepaling via satellietnavigatie. Dit component levert real-time geografische coördinaten (breedtegraad, lengtegraad, hoogte) en tijdinformatie voor het monitoringsysteem, wat essentieel is voor locatiegebaseerde toepassingen.

## Properties
- **Hardware Interface**: UART (Serial)
- **Voedingsspanning**: 3.3V
- **Stroomverbruik**: 45 mA (tracking)
- **Positienauwkeurigheid**: 2-5 meter (open lucht)
- **Opstarttijd (TTFF)**: 30-60 seconden (koude start)
- **Satellietondersteuning**: GPS, GLONASS

## Methoden
- **Initialisatie**: Serial1.begin(9600) voor UART-communicatie
- **Data lezen**: Serial1.available(), Serial1.read()
- **Coördinaat extractie**: Verwerking van NMEA-strings via TinyGPS++
- **Stroombeheer**: ESP32 diepe slaapmodus tussen metingen

## Gebruik

### Hardware Aansluiting
De NEO-8M GPS-module is direct geïntegreerd op de TTGO T-Beam printplaat. De UART-pinnen zijn verbonden met de ESP32 op pin 4 (TX) en pin 5 (RX).

### Software Configuratie
```cpp
#include <TinyGPS++.h>

TinyGPSPlus gps;

#define GPS_RX_PIN 4
#define GPS_TX_PIN 5

void setup() {
  Serial.begin(115200);
  Serial1.begin(9600, SERIAL_8N1, GPS_RX_PIN, GPS_TX_PIN);
}
```

### Data Uitlezen
```cpp
void loop() {
  while (Serial1.available() > 0) {
    char c = Serial1.read();
    
    // Parse de GPS data
    if (gps.encode(c)) {
      // Controleer of we een geldige locatie hebben
      if (gps.location.isValid()) {
        Serial.print("Latitude: ");
        Serial.println(gps.location.lat(), 6);
        Serial.print("Longitude: ");
        Serial.println(gps.location.lng(), 6);
        Serial.print("Altitude: ");
        Serial.println(gps.altitude.meters());
        
        Serial.print("Satellites in use: ");
        Serial.println(gps.satellites.value());
        
        if (gps.time.isValid()) {
          Serial.print("Time: ");
          Serial.print(gps.time.hour());
          Serial.print(":");
          Serial.print(gps.time.minute());
          Serial.print(":");
          Serial.println(gps.time.second());
        }
      }
    }
  }
}
```

### LoRaWAN Integratie
De GPS-data wordt geïntegreerd in de LoRaWAN payload zoals geïmplementeerd in de main firmware:

```cpp
// Functie om GPS data te lezen en toe te voegen aan LoRa payload
bool getGps() {
  bool newData = false;
  
  // Lees GPS data voor maximaal 1 seconde
  for (unsigned long start = millis(); millis() - start < 1000;) {
    while (Serial1.available()) {
      char c = Serial1.read();
      if (gps.encode(c)) {
        newData = true;
      }
    }
  }

  if (newData) {
    // Controleer of GPS data geldig is
    if (gps.location.isValid() && gps.location.age() < 2000 && gps.altitude.isValid()) {
      Latitude = gps.location.lat();
      Longitude = gps.location.lng();
      Altitude = gps.altitude.meters();
      gps_sats = gps.satellites.value();
      
      return true;
    } else {
      Serial.println("Geen geldige GPS data");
      return false;
    }
  } else {
    Serial.println("Geen nieuwe GPS data");
    return false;
  }
}
```

De GPS data wordt vervolgens toegevoegd aan de LoRa payload:

```cpp
        char buf[256];
        snprintf(buf, sizeof(buf), "{\"device\":\"ESP32-GPS\",\"lat\":%.6f,\"lon\":%.6f,\"alt\":%.2f,\"sats\":%d,\"ts\":%lu,\"temp\":%.2f,\"light\":\"%s\"}", lat, lon, alt, sats, now, temparatureC ,lightStr); // printe message met de gps data 

        String payload = String(buf) + "\n";                    // newline-delimited for easy parsing
        ResponseStatus s = e22ttl.sendMessage(payload); 
```

## Problemen en Oplossingen

### 1. Lange Opstarttijd (TTFF)
**Probleem**: Bij koude start kan de module tot 60 seconden nodig hebben om satellieten te vinden.
**Oplossing**:
- Module buiten plaatsen met vrij zicht op de hemel
- Wacht voldoende tijd voor eerste fix
- Houd GPS-module aan voor warme start wanneer mogelijk

### 2. Onnauwkeurige Posities
**Probleem**: Coördinaten kunnen afwijken met 10+ meter in stedelijke gebieden.
**Oplossing**:
- Wacht op voldoende satellieten (>4 voor basisnauwkeurigheid)
- Gebruik gemiddelde van meerdere metingen indien nodig
- Vermijd metingen bij slechte HDOP waarden

### 3. Stroomverbruik Optimalisatie
**Probleem**: Hoog stroomverbruik van GPS-module in continue modus.
**Oplossing**:
We maken gebruik van de diepe slaapmodus van de ESP32 tussen metingen door. De data wordt niet frequent gestuurd (elke 60+ seconden), dus er is geen hoge datafrequentie. Serial buffer overloop is daarom niet relevant. De ESP32 gaat in diepe slaap tussen metingen:

```cpp
// Configureer ESP32 diepe slaap tussen metingen
void enterDeepSleep(int sleepSeconds) {
  Serial.println("Entering deep sleep for " + String(sleepSeconds) + " seconds");
  
  // Configureer wake-up timer
  esp_sleep_enable_timer_wakeup(sleepSeconds * 1000000);
  
  // Start diepe slaap
  esp_deep_sleep_start();
}
```

### 4. Geen GPS Fix
**Probleem**: Module vindt geen satellieten of krijgt geen fix.
**Oplossing**:
- Controleer of module buiten staat met vrij zicht
- Controleer antenne-aansluiting
- Verhoog tijd voor fix-acquisitie
- Reset module indien nodig


## Configuratie Tips

### Optimalisatie voor Betrouwbaarheid
1. **Plaatsing**: Altijd buiten plaatsen met vrij zicht op de hemel
2. **Timing**: Wacht minimaal 1-2 minuten voor eerste fix
3. **Validatie**: Controleer altijd `gps.location.age()` om verse data te garanderen
4. **Satellieten**: Wacht op minimaal 4 satellieten voor betrouwbare positie

### Antenne Positie
- Plaats module met GPS-antenne naar boven gericht
- Vermijd metalen objecten in directe nabijheid
- Outdoor plaatsing geeft beste resultaten

## Zie ook
- [TTGO T-Beam Pinout Diagram](https://github.com/LilyGO/TTGO-T-Beam)
- [TinyGPS++ Library Documentation](https://github.com/mikalhart/TinyGPSPlus)
- [u-blox NEO-8M Datasheet](https://www.u-blox.com/en/product/neo-8-series)
- [ESP32 Deep Sleep Documentation](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/system/sleep_modes.html)

