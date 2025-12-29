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

```sh
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

```sh
cat /boot/firmware/config.txt
```

Aan het einde van dit bestand zal de volgende configuratie aanwezig zijn:
```sh
[all]
dtparam=uart0=on
```
Deze lijn geeft aan dat UART0 enabled is, waardoor seriële communicatie via `/dev/ttyAMA0` mogelijk wordt.

### Autostart Node-RED [Milan]

Om Node-RED automatisch te starten bij het opstarten van de Raspberry Pi en het dashboard fullscreen weer te geven, wordt gebruikgemaakt van een shellscript en een `.desktop`-bestand.

Shellscript (`nodered_autostart.sh`)
```sh
sudo systemctl stop nodered.service;
sudo systemctl start nodered.service;
sleep 5;
chromium --kiosk --password-store=basic "http://127.0.0.1:1880/dashboard"; # open in browser
```

De eerste twee commando’s herstarten Node-RED. Dit zorgt ervoor dat Node-RED zich steeds in een bekende en stabiele toestand bevindt bij het opstarten van het systeem.

Vervolgens wordt er vijf seconden gewacht om te garanderen dat Node-RED volledig is opgestart voordat het dashboard wordt geopend.

Ten slotte wordt Chromium gestart in kioskmodus, waarbij het Node-RED dashboard automatisch fullscreen op het aangesloten beeldscherm wordt weergegeven. Hierdoor is het systeem geschikt voor visualisatie- of monitoringtoepassingen.

Autostart-configuratie (`~/.config/autostart/red.desktop`)
```sh
[Desktop Entry]
Type=Application
Name=Node-RED Autostart
Exec=<script-location>/nodered_autostart.sh
X-GNOME-Autostart-enabled=true"http://127.0.0.1:1880";
```

Het .desktop-bestand in `~/.config/autostart/` zorgt ervoor dat het shellscript automatisch wordt uitgevoerd zodra de grafische desktopomgeving van de Raspberry Pi is geladen.

De parameter `Exec` verwijst naar de locatie van het shellscript `nodered_autostart.sh`, terwijl `X-GNOME-Autostart-enabled=true` aangeeft dat deze applicatie automatisch wordt opgestart.

#### Controle van autostart:
```sh
chmod +x red.desktop;
sudo reboot;
```
Door het .desktop-bestand uitvoerbaar te maken en het systeem te herstarten, kan de correcte werking van de autostart worden gevalideerd.

Na het herstarten start Node-RED automatisch en wordt het dashboard zonder gebruikersinteractie in kioskmodus weergegeven.

### Configuratie & Communicatie met E220 [Milan]

Voor de configuratie en werking van de LoRa-module wordt UART gebruikt in combinatie met M0, M1 en AUX. Door deze signalen correct aan te sturen kan dynamisch worden gewisseld tussen configuratiemodus (AT-commands) en normale transmissiemodus.

Om het wisselen tussen configuratie- en transmissiemodus mogelijk te maken, worden M0 & M1 softwarematig aangestuurd. AUX wordt gebruikt als indicator wanneer de module bezig is.

Pseudocode voor configuratie van receiver:
1. M0 & M1 : 3.3 V / HIGH				      ( Configuration / Sleep mode )
2. Wacht tot AUX uitgang = HIGH 			( E220 niet bezig )
3. Verzend @-command					( UART )
4. Wacht tot AUX = HIGH 			
5. Lees alles wat E220 beantwoordt		( UART )
6. Stap 3 OF exit config M0 & M1 : GND	( Transmission mode )
7. Wacht tot AUX = HIGH
8. Ontvang LoRa berichten …

**Zie ook manual E220-xxxTxxx : 5.2 AUX Timing, 7. AT Commands**

Het volledig Python-codevoorbeeld voor de ontvanger is terug te vinden in:  
`Receiver/reset_config_pi5/config+reset.py`

```python
# ---------------- Configuration ----------------
SERIAL_PORT = '/dev/ttyAMA0'
BAUDRATE = 9600
TIMEOUT = 1

AUX_PIN = 18
M0_PIN = 21
M1_PIN = 20

# ---------------- Setup GPIO ----------------
GPIO.setmode(GPIO.BCM)
# ...

def wait_aux_ready():
    """Wait until AUX pin goes HIGH (LoRa ready)."""
    while GPIO.input(AUX_PIN) == 0:
        time.sleep(0.01)

def enter_at_mode():
    """Enter AT command mode (M0=1, M1=1)."""
    GPIO.output(M0_PIN, GPIO.HIGH)
    GPIO.output(M1_PIN, GPIO.HIGH)
    time.sleep(0.1)
    wait_aux_ready()
    print("Entered AT command mode")

def exit_at_mode():
    """Return to normal transmission mode (M0=0, M1=0)."""
    GPIO.output(M0_PIN, GPIO.LOW)
    GPIO.output(M1_PIN, GPIO.LOW)
    time.sleep(0.1)
    wait_aux_ready()
    print("Exited AT command mode")

def open_serial():
    ser = serial.Serial(SERIAL_PORT, BAUDRATE, timeout=TIMEOUT)
    print(f"Connected to {SERIAL_PORT} at {BAUDRATE}bps")
    wait_aux_ready()
    return ser

def send_at_command(ser, command):
    """Send AT command and return response."""
    full_cmd = (command + '\r\n').encode('utf-8')
    wait_aux_ready()
    ser.write(full_cmd)
    wait_aux_ready()
    time.sleep(0.1)
    response = ser.read_all().decode(errors='ignore').strip()
    return response

def receive_data(ser):
    """Continuously read data from LoRa module."""
    print("Listening for incoming LoRa data...")
    try:
        while True:
            if ser.in_waiting:
                data = ser.read(ser.in_waiting)
                try:
                    text = data.decode('utf-8', errors='replace')
                except:
                    text = repr(data)
                print("AUX:", GPIO.input(AUX_PIN))
                print("Received:", text)
            time.sleep(0.1)
    except KeyboardInterrupt:
        print("Stopping reception...")
        
def main():
    ser = open_serial()
    enter_at_mode()

    commands = [
        "AT+UART=?",
        "AT+RATE=?",
        "AT+PACKET=?",
        "AT+ADDR=?",
        "AT+CHANNEL=?",
        "AT+MODE=?",
        "AT+TRANS=?",
    ]

    for cmd in commands:
        print(f"\nSending: {cmd}")
        response = send_at_command(ser, cmd)
        print(f"Response: {response}")
        time.sleep(0.5)
    exit_at_mode()
    receive_data(ser)
    ser.close()
```

#### Belangrijke functies in de code

- `enter_at_mode()` / `exit_at_mode()`  
    Wisselen tussen configuratie- en transmissiemodus via GPIO. (at-mode = configuratiemodus)
- `wait_aux_ready()`  
    Blokkering wanneer de module bezig is om timingproblemen te vermijden.
- `send_at_command()`  
    Verzenden en uitlezen van AT-commando’s via UART in configuratiemodus.
- `receive_data()`  
    Uitlezen van ontvangen in transmissiemodus.