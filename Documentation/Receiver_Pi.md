## 2. Ontvanger (Raspberry Pi 5)

### GPIO

De communicatie tussen de Raspberry Pi 5 en de LoRa E220-module verloopt via UART. Volgende tabel geeft een overzicht van de gebruikte GPIO-aansluitingen en hun functie.

| LoRa E220 | Raspberry Pi       | Opmerkingen /dev/ttyAMA0                          |
| --------- | ------------------ | ------------------------------------------------- |
| VCC       | 3.3 V - 5 V        | Voeding                                           |
| GND       | GND                | Aarde                                             |
| TX        | UART RX (GPIO 15 ) | Communicatie UART                                 |
| RX        | UART TX (GPIO 14)  | Communicatie UART                                 |
| M0        | GND  / GPIO 21     | Normale modus (GND) /  Configuratie modus (+3.3V) |
| M1        | GND  / GPIO 20     | Normale modus (GND) /  Configuratie modus (+3.3V) |
| AUX       | GPIO 18            | Busy status E220                                  |

Een LCD-scherm wordt aangesloten via het I2C-protocol om ontvangen gegevens visueel weer te geven.

| LCD | Raspberry Pi | Opmerkingen                    |
| --- | ------------ | ------------------------------ |
| SCL | SCL (GPIO 3) | Communicatie I2C: Serial clock |
| SDA | SDA (GPIO 2) | Communicatie I2C: Serial Data  |
| GND | GND          | Aarde                          |
| VCC | 5 V          | Voeding                        |

Een drukknop wordt gebruikt om de Raspberry Pi veilig in en uit te schakelen. Daarnaast geeft een LED de operationele status van het systeem en NodeRed weer.

| Button  ON/OFF | Raspberry Pi | Opmerkingen (Normaal open knop)                                                                                                     |
| -------------- | ------------ | ----------------------------------------------------------------------------------------------------------------------------------- |
| LED +          | GPIO 17      | Statusindicatie: Pi actief en Node-RED operationeel (met serieweerstand)                                                            |
| LED -          | GND          | Aarde LED                                                                                                                           |
| Button 0       | GPIO x & J2  | GPIO x voor softwarematige uitschakeling, J2 voor hardware power control (J2 bevindt zich tussen de RTC-batterij en de USB-C-poort) |
| Button 1       | GND J2/...   | Aarde Button                                                                                                                        |


![GPIO Raspberry Pi](/GPIO's/Raspberry-Pi-5-Pinout--189012982.jpg)    

### Enable UART & I2C [Milan]

Om seriële communicatie via UART en I2C mogelijk te maken, dienen deze geactiveerd te worden via de `Interfacing Options` van raspi-config.

```
sudo raspi-config
```

![[Afbeeldingen/raspi-config.png]]

Configureer de volgende opties:
- **Serial Port**
	- Login shell over serial: Disabled
	- Serial port hardware: Enabled
- **I2C**: Enabled

Bevestig de wijzigingen en herstart de Raspberry Pi.

#### Controle via configuratiebestand

Om te testen dat seriële communicatie via UART geactiveerd is, kan in dit configuratiebestand van de Raspberry Pi gekeken worden.

```
cat /boot/firmware/config.txt
```

Aan het einde van dit bestand zal de volgende configuratie aanwezig zijn:
```
[all]
dtparam=uart0=on
```
Deze lijn geeft aan dat UART0 enabled is, waardoor seriële communicatie via `/dev/ttyAMA0` mogelijk wordt.

### Autostart Node-RED [Milan]

Om Node-RED automatisch te starten bij het opstarten van de Raspberry Pi en het dashboard fullscreen weer te geven, wordt gebruikgemaakt van een shellscript en een `.desktop`-bestand.

Shellscript (`nodered_autostart.sh`)
```
sudo systemctl stop nodered.service;
sudo systemctl start nodered.service;
sleep 5;
chromium --kiosk --password-store=basic "http://127.0.0.1:1880/dashboard"; # open in browser
```

De eerste twee commando’s herstarten Node-RED. Dit zorgt ervoor dat Node-RED zich steeds in een bekende en stabiele toestand bevindt bij het opstarten van het systeem.

Vervolgens wordt er vijf seconden gewacht om te garanderen dat Node-RED volledig is opgestart voordat het dashboard wordt geopend.

Ten slotte wordt Chromium gestart in kioskmodus, waarbij het Node-RED dashboard automatisch fullscreen op het aangesloten beeldscherm wordt weergegeven. Hierdoor is het systeem geschikt voor visualisatie- of monitoringtoepassingen.

Autostart-configuratie (`~/.config/autostart/red.desktop`)
```
[Desktop Entry]
Type=Application
Name=Node-RED Autostart
Exec=<script-location>/nodered_autostart.sh
X-GNOME-Autostart-enabled=true"http://127.0.0.1:1880";
```

Het .desktop-bestand in `~/.config/autostart/` zorgt ervoor dat het shellscript automatisch wordt uitgevoerd zodra de grafische desktopomgeving van de Raspberry Pi is geladen.

De parameter `Exec` verwijst naar de locatie van het shellscript `nodered_autostart.sh`, terwijl `X-GNOME-Autostart-enabled=true` aangeeft dat deze applicatie automatisch wordt opgestart.

#### Controle van autostart:
```
chmod +x red.desktop;
sudo reboot;
```
Door het .desktop-bestand uitvoerbaar te maken en het systeem te herstarten, kan de correcte werking van de autostart worden gevalideerd.

Na het herstarten start Node-RED automatisch en wordt het dashboard zonder gebruikersinteractie in kioskmodus weergegeven.

