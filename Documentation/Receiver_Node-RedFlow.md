# Node-RED flow — LoRa-ontvanger

## Inleiding

Deze Node-RED-flow leest LoRa-data in op de Raspberry Pi, verwerkt de ontvangen JSON-pakketten en stuurt zowel het dashboard als de LCD-weergave aan.

De flow vormt de centrale schakel tussen de LoRa-ontvanger en de eindgebruiker en zorgt voor een betrouwbare verwerking en visualisatie van alle binnenkomende data. Naast het dashboard wordt ook de volledige logica voor het aansturen van het LCD-scherm binnen deze Node-RED-flow afgehandeld.

De belangrijkste onderdelen van de flow zijn:

* `LoRa Input` – ontvangt ruwe data via UART
* `parse_lora_json` – valideert en verrijkt de JSON-data
* `update_device_cache` – houdt actieve devices en hun laatste data bij
* `transform_for_lcd` – vertaalt devive data naar LCD schermformaat en regelt schermrotatie
* `handle_device_actions` – verwerkt acties vanuit de gebruikersinterface
* `template (UI)` – visualiseert data in het dashboard
De function nodes worden in de volgende secties afzonderlijk en gedetailleerd uitgewerkt.

---

## Overzicht van de flow

Onderstaande afbeelding toont de volledige Node-RED-flow van de ontvanger. De LoRa-data komt via UART de flow binnen, daarna worden de JSON-pakketten geparset en in een device-cache opgeslagen. Vervolgens worden het dashboard, het LCD-scherm en de acties vanuit de UI afgehandeld.

> **Let op:** onderstaande afbeelding toont nog de **oude** Node‑RED‑flow.  


![Node-RED flow ontvanger (oude versie)](../Afbeeldingen/NodeRedFlow.jpg)
---

## LoRa Input

- Deze node vormt het startpunt van de flow en ontvangt ruwe data via de UART-verbinding met de E220 LoRa-module.

## parse_lora_json
- Uitleg: Deze functie verwerkt LoRa JSON berichten, haalt belangrijke data zoals locatie, apparaat-ID, temperatuur en voegt in een leesbare string.
```js
// Ik controleer of msg.payload een string is en verwijder spaties aan het begin en einde
let raw = (typeof msg.payload === "string") ? msg.payload.trim() : String(msg.payload);
// Ik begin met een object waarin ik altijd de ruwe data opsla
let data = { raw };
try {
  // Ik probeer de string te parsen naar een JSON-object
  const obj = JSON.parse(raw);
  // Ik voeg alle velden uit het JSON-object toe aan mijn data-object
  Object.assign(data, obj);
} catch (e) {
  // Als het parsen mislukt, geef ik een waarschuwing en bewaar ik alleen de ruwe data
  node.warn("JSON parse failed, keeping raw only: " + e);
}
// Ik haal losse velden op, of geef een standaardwaarde als ze ontbreken
const device = data.device !== undefined ? String(data.device) : "Unknown";
const lat    = data.lat    !== undefined ? String(data.lat)    : "??";
const lon    = data.lon    !== undefined ? String(data.lon)    : "??";
const alt    = data.alt    !== undefined ? String(data.alt)    : "??";
const sats   = data.sats   !== undefined ? String(data.sats)   : "??";
// Ik maak een leesbare locatie-string
data.location_str =
  "Device: " + device +
  " | Lat: " + lat + ", Lon: " + lon +
  " | Alt: " + alt + " m | Sats: " + sats;
// Ik voeg temperatuur als string toe als die aanwezig is
if (typeof data.tempC === "number" && !isNaN(data.tempC)) {
  data.tempC_str = data.tempC.toFixed(2);
}
// Ik zet het resultaat terug in msg.payload
msg.payload = data;
return msg;
```

## update_device_cache
- Uitleg: deze functie gebruik ik als ' database ' hier beheer ik een lijst van alle actieve devices. Ik voeg nieuwe devices toe, werk bestaande bij en verwijder oude of onbekende devices uit het geheugen. Zo houd ik altijd een actuele lijst bij voor de rest van de flow.
```js
// Ik haal het binnengekomen device-object op
let d = msg.payload || {};
// Ik noteer het huidige tijdstip
const now = Date.now();
// Ik bewaar het huidige device-object voor eventueel debuggen
msg.current = d;
// Ik bepaal hoe lang een device als 'actief' mag blijven zonder update
const STALE_MS = 5 * 60 * 1000;
// Ik bepaal de naam van het device
let name = (d.deviceId || d.device || "").trim();
// Ik haal de bestaande device-cache op uit het geheugen
let devices = flow.get("devices") || {};
// Ik verwijder oude of onbekende devices uit de cache
Object.keys(devices).forEach(id => {
  const dev = devices[id];
  if (!id || id.toLowerCase() === "unknown") {
    delete devices[id];
    return;
  }
  if (dev && dev.lastSeen && (now - dev.lastSeen) > STALE_MS) {
    delete devices[id];
    return;
  }
});
// Als het huidige device geen geldige naam heeft, sla ik alleen de opgeschoonde cache op en stop ik
if (!name || name.toLowerCase() === "unknown") {
  flow.set("devices", devices);
  return null;
}
// Ik voeg het nieuwe of bijgewerkte device toe aan de cache
devices[name] = {
  ...(devices[name] || {}),
  ...d,
  deviceId: name,
  lastSeen: now
};
// Ik sla de cache weer op
flow.set("devices", devices);
// Ik zet alle devices als array in msg.payload voor de volgende stap
msg.payload = Object.values(devices);
return msg;
```

---

## transform_for_lcd
- Uitleg: In deze functie maak ik van de device-data een lijst van tekstregels die geschikt zijn voor het LCD-scherm. Ik zorg dat de informatie netjes past, meerdere devices kunnen roteren, en het formaat past zich aan het scherm aan.
```js
// Ik haal de lijst met devices op uit msg.payload of uit het geheugen
const devices = Array.isArray(msg.payload) ? msg.payload : (flow.get('devices') ? Object.values(flow.get('devices')) : []);
// Ik bepaal het aantal kolommen en rijen van het LCD-scherm
const cols = flow.get('lcd_columns') || 16;
const rows = flow.get('lcd_rows') || 2;
const rotateMs = flow.get('lcd_rotate_ms') || 4000;

// Ik maak een hulpfunctie om tekst op maat te maken
function padOrTrim(s, len) {
  s = String(s ?? '');
  if (s.length > len) return s.slice(0, len);
  return s + ' '.repeat(len - s.length);
}

// Ik bepaal hoeveel devices er tegelijk op het scherm passen
let devicesPerScreen = 1;
if (cols >= 20 && rows >= 4) devicesPerScreen = 2;
if (cols >= 20 && rows === 2) devicesPerScreen = 1;

// Ik maak van elk device een object met de juiste velden voor het scherm
const items = devices.map(d => {
  const id = d.deviceId || d.device || 'Unnamed';
  const temp = Number.isFinite(d.tempC) ? d.tempC.toFixed(1) + '°C' : (d.tempC_str || '—');
  const batt = Number.isFinite(d.battery) ? Math.round(d.battery) + '%' : '';
  const sats = Number.isFinite(d.sats) ? String(d.sats) + 'sat' : '';
  return { id, temp, batt, sats };
});

// Ik bouw schermen op, elk scherm bevat regels voor het LCD
const screens = [];
for (let i = 0; i < items.length; i += devicesPerScreen) {
  const chunk = items.slice(i, i + devicesPerScreen);
  const lines = [];
  chunk.forEach(item => {
    lines.push(padOrTrim(item.id, cols));
    let right = item.temp;
    if (item.batt) right += ' ' + item.batt;
    if (right.length > cols && item.sats) {
      const alt = item.temp + ' ' + item.sats;
      if (alt.length <= cols) right = alt;
    }
    lines.push(padOrTrim(right, cols));
    if (rows >= 4) {
      lines.push(padOrTrim(item.sats || '', cols));
      lines.push(padOrTrim(item.batt || '', cols));
    }
  });
  while (lines.length < rows) lines.push(' '.repeat(cols));
  screens.push(lines.slice(0, rows));
}

// Als er geen devices zijn, toon ik een placeholder
if (screens.length === 0) {
  screens.push([ padOrTrim('No devices', cols), padOrTrim('', cols) ]);
}

// Ik regel de rotatie van de schermen
let idx = flow.get('lcd_screen_index') || 0;
const last = flow.get('lcd_last_rotate') || 0;
const now = Date.now();
if (now - last >= rotateMs) {
  idx = (idx + 1) % screens.length;
  flow.set('lcd_screen_index', idx);
  flow.set('lcd_last_rotate', now);
}
// Ik laat externe triggers toe om direct door te draaien
if (msg.forceNext === true) {
  idx = (idx + 1) % screens.length;
  flow.set('lcd_screen_index', idx);
  flow.set('lcd_last_rotate', now);
}

// Ik zet de regels om naar het juiste formaat voor het LCD-node
const screen = screens[idx];
const out = screen.map(line => {
  return { clear: true, text: line, alignment: "left" };
});

// Ik voeg debug-informatie toe voor het testen
msg._lcd_debug = {
  index: idx,
  totalScreens: screens.length,
  cols, rows
};
msg.payload = out;
return msg;
```

---

## handle_device_actions
- Uitleg: In deze functie verwerk ik acties vanuit de gebruikersinterface, zoals het verwijderen van een device. Als er een delete-actie binnenkomt, haal ik het juiste device uit de cache en stuur ik de bijgewerkte lijst terug.
```js
// Ik haal de actie op uit het bericht
const p = msg.payload || {};
// Ik controleer of het een delete-actie is en of er een deviceId is opgegeven
if (p.action === "delete" && p.deviceId) {
  // Ik haal de huidige device-cache op
  let devices = flow.get("devices") || {};
  // Ik verwijder het opgegeven device uit de cache
  delete devices[p.deviceId];
  // Ik sla de aangepaste cache weer op
  flow.set("devices", devices);
  // Ik stuur de nieuwe lijst van devices terug
  msg.payload = Object.values(devices);
  return msg;
}
// Als het geen geldige actie is, doe ik niets
return null;
```

---

## template (UI)

- Uitleg: In deze template visualiseer ik de actuele device-data uit de device_cache op het dashboard. Hieronder beschrijf ik de belangrijkste logica:
  
  - Ik haal de lijst met devices op uit de device_cache (via msg.payload, die gevuld wordt door de update_device_cache functie).
  - Voor elk device toon ik een kaart met de naam, status (online, idle, offline), temperatuur, GPS-positie.
  - Bovenaan het dashboard toon ik een samenvatting van het aantal online/idle/offline devices, gemiddelde temperatuur, zwakke GPS en hoogste hoogte.
  - Met knoppen op elke device-kaart kan ik een device verwijderen (actie wordt via handle_device_actions verwerkt) of coördinaten kopiëren.
  - De interface past zich automatisch aan het aantal devices en hun status aan.
  - Alle logica voor het bepalen van status, labels en acties staat in de methods van de template (zie /Node-Red/[template.html] voor details).

### Voorbeeld code snippet:

```html
<!-- Ik haal de lijst met devices uit de device_cache via msg.payload en toon bijvoorbeeld de device id en gps enz... -->
<div v-for="d in msg.payload">
  <span>Device: {{ d.deviceId || d.device || 'Unnamed device' }}</span>
  <span>Status: {{ statusLabel(d.lastSeen) }}</span>
  <span>Last seen: {{ formatLastSeen(d.lastSeen) }}</span>
  <span>Temperature: {{ d.tempC_str ? d.tempC_str : (Number.isFinite(d.tempC) ? d.tempC.toFixed(2) : '—') }} °C</span>
  <span>GPS: {{ d.lat }}, {{ d.lon }}</span>
  <span>Altitude: {{ Math.round(d.alt) }} m</span>
  <span>Satellites: {{ d.sats }}</span>
  <span>Packet age: {{ d.ts }}</span>
  <!-- knoppen -->
  <button @click="removeDevice(d)">Verwijder</button>
  <button @click="copyCoordinates(d)">Kopieer coördinaten</button>
</div>
```
---

*Gemaakt door Anish*
