# BMW CarData Streaming MQTT Bridge (v1.27.1)

[![Buy Me A Coffee](https://cdn.buymeacoffee.com/buttons/v2/default-yellow.png)](https://www.buymeacoffee.com/bausi2k)
[![License: CC BY-NC-SA 4.0](https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-nc-sa/4.0/)

🇬🇧 [English Version below](#english) | 🇦🇹 [Deutsche Version unten](#deutsch)

---

<a name="english"></a>
## 🇬🇧 BMW CarData Streaming MQTT Bridge

This project acts as a robust, long-running bridge service connecting the official **BMW CarData Streaming API** with your local MQTT broker. It receives real-time vehicle data (Push/Streaming) and forwards it to your home automation system (e.g., Node-RED, Home Assistant, Grafana).

It handles the entire OAuth2 authentication lifecycle, including automatic token refreshing, ensuring a maintenance-free operation.

### ✨ Features
* **Web-UI Dashboard:** Premium glassmorphic interface built strictly on the **CGDESIGN (v1.0.0)** design language (classic BMW blue accents, dynamic blur glass effect, consistent corner styling) to view live car telemetry, customize card sizes, edit configuration (.env), and stream live logs.
* **Light & Dark Mode:** Switch themes at any time via the toggle in the header. Your choice is stored in `localStorage`, and on first load the bridge follows your operating system's preference (`prefers-color-scheme`). Map tiles adapt automatically.
* **BMW CarData API Integration:** Fetches the official vehicle image and a full vehicle profile — model, series, body, colour, build date, drivetrain, head unit, SIM status and the complete option-code list. `basicData` is fetched daily anyway; until v1.13.0 only two of its 22 fields were shown. The nominal capacity carries its unit since v1.16.0 (`210.6 Ah`), but **no energy is derived from it**: that needs the pack voltage, which BMW ships in no field and which is not 400 V everywhere — the Neue Klasse runs on 800 V, where the same arithmetic would show half the capacity. The usable energy comes from the data stream instead (`batteryManagement.maxEnergy`), measured rather than calculated, and costs no API call.
* **Overview Split Map:** Optional Leaflet.js real-time navigation map embedded side-by-side with the car image (2/3 image, 1/3 map) with dynamic GPS and rotated vector heading markers.
* **Long-Term Route History:** Continuous SQLite logging of location coordinates with an interactive historical path map tab (time-filtered, follows the selected theme). The track is split into segments wherever the car stopped reporting: a dashed line marks a stretch where the route is unknown, rather than pretending the car drove straight through. Positions no car could have reached are left out of the drawing — the database keeps every recorded point.
* **Daily Budget Aware:** BMW allows 50 API calls per day, reset at 00:00 UTC. Charging sessions are **archived in SQLite**, so only what is new gets fetched — typically a single page instead of eight to ten. Measured on the production system: one morning cost 29 calls before, about three after. Everything else is cached and **survives a container restart**; the stored age is the time of the fetch, not of the load, so a restart cannot silently extend the cache lifetime. Concurrent requests for the same data share a single fetch — two overlapping ones once cost 28 calls for what a user experienced as one click. If BMW aborts mid-pagination, the pages already paid for are kept and the gap is reported instead of discarding everything.
* **Charging History:** A dedicated tab lists your charging sessions — start, location, energy drawn, state of charge from/to, time on the cable and the power measured **while charging**. That last one matters: energy divided by total plug-in time collapses to a fraction once the car sits full on the cable, so the figure is taken from BMW's charging blocks instead. Filter by a free date-and-time range, by AC or DC, and by peak power — all locally, none of it costs an API call. **AC and DC are told apart** from the peak block power; BMW ships no field for it, so this is a derivation and labelled as one. Sessions where nothing reached the battery can be hidden with the **0 kWh** toggle. The summary always follows the visible rows, so the average becomes "per actual charge" rather than "per plug-in". **Every column sorts** on a click — state of charge by the difference between start and end, and power by the value actually shown, so imported rows with a derived figure take part instead of sinking to the bottom. Missing values always sort last, in both directions. The archive grows **beyond BMW's 45-day limit** — older ranges the API refuses stay available once stored.
* **Import from the BMW app export:** The app exports the charging history as a monthly `.xlsx`. It carries five fields the CarData API does not have — cost, tariff, charging location name, address and label — and it reaches back as far as your exports go, which is the only way to build the archive backwards past the API's 45 days. Import costs no API call. **An imported value never overwrites a measured one:** BMW itself calls these figures forecasts in the file's own footnote — the export says `~ 12 kWh` where the API says `22.979991912841797` for the same session. Values from the export are marked with an asterisk, and sessions that exist only there get no AC/DC classification, because the export carries no charging blocks and an invented one would be worse than none.
* **Vehicle List:** Shows every vehicle mapped to your BMW account and your role for it. Useful beyond curiosity: CarData only returns data for vehicles where you are the **primary user**, which is a common cause of `CU-104` errors.
* **API Budget Awareness:** BMW allows 50 API calls per day (reset at 00:00 UTC) and reports the limit as HTTP 403 — easily mistaken for a permission problem. The bridge counts its calls, explains all documented error codes in plain language, and stops asking once the budget is spent.
* **Real-time Streaming:** Connects to BMW's MQTT interface via WebSockets/MQTT.
* **Robust Authentication:** Implements the OAuth2 Device Code Flow.
* **Auto-Healing:** Refreshes access tokens before they expire — checked every 5 minutes, renewed with 10 minutes to spare. The margin deliberately covers two check cycles, so a failed refresh gets a second attempt while the token is still alive; a single-attempt setting cost the stream on the first hiccup.
* **Watchdog Reconnect:** Monitors data traffic and automatically reconnects if the stream stalls.
* **Outage Backoff:** When BMW's REST API reports a temporary fault, further calls pause instead of piling up failed attempts — 5, 15, 30 then 60 minutes while it lasts, cleared by the first successful call. A **„Try anyway“** button overrides it for a single attempt. The pause is stored in the database, so a restart does not lift it. The live MQTT stream is unaffected.
* **Dockerized:** Available as a pre-built Multi-Arch Image (amd64/arm64) via GitHub Container Registry.
* **Dynamic Topics:** Flattens complex JSON data into clean MQTT topics (e.g., `home/bmw/live/vehicle/mileage`).
* **Multi Car Mode:** Set `LOCAL_MQTT_APPEND_VIN=true` in your `.env` to append each vehicle's VIN to the base topic (e.g. `home/bmw/live/WBA…/vehicle/mileage`). This keeps several cars apart on a single broker.

### ⚠️ Acknowledgements & Credits

**This project is built upon the foundational work of:**
👉 **[whi-tw/bmw-cardata-streaming-poc](https://github.com/whi-tw/bmw-cardata-streaming-poc)**

The core Python client logic (`lib/bmw_cardata.py`) responsible for the protocol implementation and authentication flow is taken from that repository. A huge thank you to the author for reverse-engineering the API!

---

### 🚀 Installation (Docker Compose)

The easiest way to run this bridge is using the pre-built Docker image. You do not need to clone the code or build it manually.

#### 1. Prerequisites & Credentials

To use this bridge, you need specific credentials from the BMW Developer Portal.

* **Client ID:** Register and obtain your Client ID here:  
    [BMW CarData - Technical Registration](https://bmw-cardata.bmwgroup.com/customer/public/api-documentation/Id-Technical-registration)
* **MQTT Username:** Find your specific MQTT Username here:  
    [BMW CarData - Streaming Documentation](https://bmw-cardata.bmwgroup.com/customer/public/api-documentation/Id-Streaming)

#### 2. Configuration Files

Create a folder for the project and add the following files:

**`.env`** (Configuration)
*⚠️ Security Warning: Never share this file or commit it to GitHub!*
```ini
# BMW Config
# Insert the credentials obtained from the BMW Portal links above
CLIENT_ID=YOUR_BMW_CLIENT_ID
MQTT_USERNAME=YOUR_BMW_MQTT_USERNAME

# Local MQTT Broker
LOCAL_MQTT_URL=192.168.1.xxx
LOCAL_MQTT_PORT=1883
LOCAL_MQTT_USER=your_local_user
LOCAL_MQTT_PASS=your_local_password
LOCAL_MQTT_BASETOPIC=home/bmw/data
LOCAL_MQTT_APPEND_VIN=false #Optional for multicar Instanze: VIN a (z.B. home/bmw/live/WBA.../...)

# Logs
LOG_LEVEL=INFO

# Data retention (optional, since v1.8.9)
# How long telemetry values are kept in the SQLite database (default: 30 days)
TELEMETRY_RETENTION_DAYS=30
# How long the location history is kept. 0 = unlimited (default).
# Note: the location history is a complete movement profile.
LOCATION_RETENTION_DAYS=0
# Hours between two cleanup runs (default: 24)
MAINTENANCE_INTERVAL_HOURS=24

# Location history (optional)
# Since v1.13.0 latitude and longitude are paired by BMW's own measurement
# time, so both halves must come from the same reading. 0 means exact.
# Raise it only if the log shows valid pairs being discarded.
GPS_MAX_MEASUREMENT_DELTA=0
# Fallback for the rare case that BMW ships no measurement time: maximum gap
# in seconds between the ARRIVAL of latitude and longitude (default: 30).
GPS_MAX_TIMESTAMP_DELTA=30

# Map tiles (optional, since v1.8.10)
# Tile requests carry your vehicle's position to whichever provider is set
# here. The default is OpenStreetMap. Point this at your own tile server if
# you would rather not share that.
# MAP_TILE_URL=http://192.168.1.50:8080/tiles/{z}/{x}/{y}.png
# MAP_TILE_ATTRIBUTION=My own tile server
````

**`vehicle.json`** (Your car's VIN)

```json
{
  "vin": "WBA............."
}
```

**Initialize configuration directories & files** (Required for Docker volumes):

```bash
touch bmw_tokens.json   # leave it empty! The bridge fills it during first-run authentication
mkdir -p data logs
```

#### 3\. Create `docker-compose.yml`

```yaml
services:
  bmw-bridge:
    image: ghcr.io/bausi2k/bmw-mqtt-bridge:latest
    
    container_name: bmw-bridge
    restart: unless-stopped
    
    ports:
      - "${WEB_PORT:-8999}:${WEB_PORT:-8999}"
    
    env_file:
      - .env
    
    volumes:
      - ./bmw_tokens.json:/app/bmw_tokens.json
      - ./vehicle.json:/app/vehicle.json
      - ./logs:/app/logs
      - ./.env:/app/.env
      - ./data:/app/data
      
    environment:
      - TZ=Europe/Vienna
```

#### 4\. Start the Service

```bash
docker compose up -d
```

#### 5\. Authentication (First Run)

1.  Check the logs immediately after starting: `docker compose logs -f`
2.  You will see a URL provided by BMW.
3.  Open the URL in your browser and log in with your BMW ID to authorize the application.
4.  The script will automatically receive the tokens, save them to `bmw_tokens.json`, and start streaming.

-----

### ⚖️ License

This project is licensed under **CC BY-NC-SA 4.0**.

  * ✅ **Allowed:** Private use, modification, and sharing.
  * ❌ **Forbidden:** Commercial use or selling of this software.

See the [LICENSE](https://www.google.com/search?q=LICENSE) file for details.

### 🤝 Credits

**\#kiassisted** 🤖
This project was created with the assistance of AI.
Code architecture, logic, and documentation support provided by Gemini.

-----

-----

\<a name="deutsch"\>\</a\>

## 🇦🇹 BMW CarData Streaming MQTT Bridge

Dieses Projekt dient als stabile Brücke zwischen der offiziellen **BMW CarData Streaming API** und deinem lokalen MQTT-Broker. Es empfängt Fahrzeugdaten in Echtzeit (Push/Streaming) und leitet sie an dein Smart Home System weiter (z.B. Node-RED, Home Assistant, Grafana).

Der Service kümmert sich vollautomatisch um die OAuth2-Authentifizierung und das Erneuern der Tokens, sodass ein wartungsfreier Dauerbetrieb möglich ist.

### ✨ Funktionen
* **Web-UI Dashboard:** Edles Glassmorphism-Interface basierend auf der **CGDESIGN (v1.0.0)** Designvorgabe (klassische BMW-blaue Akzente, dynamischer Glasunschärfe-Effekt, einheitlich abgerundete Ecken) zur Anzeige von Live-Fahrzeugdaten, anpassbaren Cards, Konfigurations-Editor (.env) und Live-Log-Streaming.
* **Hell- & Dunkelmodus:** Umschaltbar über den Toggle im Header. Die Auswahl bleibt im `localStorage` erhalten, beim ersten Laden wird das Farbschema des Betriebssystems (`prefers-color-scheme`) übernommen. Die Kartenkacheln passen sich automatisch an.
* **BMW CarData API Integration:** Lädt das offizielle Fahrzeugbild und einen vollständigen Steckbrief — Modell, Baureihe, Karosserie, Farbe, Baudatum, Antrieb, Head-Unit, SIM-Status und die komplette Sonderausstattungsliste. `basicData` wird ohnehin täglich abgerufen; bis v1.13.0 wurden davon zwei von 22 Feldern angezeigt.
* **Geteilte Übersichtskarte:** Bindet optional eine Leaflet.js-Echtzeitkarte direkt neben dem Fahrzeugbild ein (2/3 Bild, 1/3 Karte) mit dynamischer GPS- und rotierter SVG-Richtungsanzeige.
* **Langzeit-Routenhistorie:** Protokolliert alle GPS-Koordinaten in einer permanenten SQLite-Tabelle und visualisiert die Fahrtwege in einem interaktiven Kartentab ("Standortverlauf") mit Zeitraum-Filter, passend zum gewählten Farbschema. Der Verlauf wird überall dort in Abschnitte getrennt, wo das Fahrzeug nichts mehr gemeldet hat: Eine gestrichelte Linie zeigt an, dass die gefahrene Strecke unbekannt ist, statt eine Gerade zu erfinden. Positionen, die kein Auto erreicht haben kann, werden nicht gezeichnet — in der Datenbank bleibt jeder aufgezeichnete Punkt erhalten.
* **Ladeverlauf:** Ein eigener Tab listet die Ladevorgänge – Beginn, Ort, geladene Energie, Ladestand von/bis, Zeit am Kabel und die Leistung **während des Ladens**. Letzteres ist der Unterschied: Energie geteilt durch die gesamte Steckzeit fällt auf einen Bruchteil, sobald das Auto voll am Kabel steht; der Wert stammt deshalb aus den Ladeblöcken von BMW. Zeiträume 7/14/30/45 Tage, wobei die kürzeren lokal filtern und keinen API-Abruf kosten. **AC und DC werden unterschieden**, abgeleitet aus der Spitzenleistung der Blöcke – BMW liefert dafür kein Feld, die Angabe ist entsprechend gekennzeichnet. Ein Klick auf eine Zeile zeigt Spitzenleistung, tatsächliche Ladezeit, Anzahl der Ladeblöcke, Kilometerstand und Vorkonditionierung. Wird einmal täglich über die BMW-Schnittstelle `chargingHistory` abgerufen.
* **Fahrzeugliste:** Zeigt alle dem BMW-Konto zugeordneten Fahrzeuge und die eigene Rolle. Praktisch relevant: CarData liefert Daten ausschließlich für Fahrzeuge, bei denen man **Hauptnutzer** ist – eine häufige Ursache für `CU-104`-Fehler.
* **Bewusster Umgang mit dem API-Limit:** BMW erlaubt 50 Abrufe pro Tag (Reset um 00:00 UTC) und meldet das Limit als HTTP 403 – leicht mit einem Berechtigungsproblem zu verwechseln. Die Bridge zählt ihre Aufrufe mit, erklärt alle dokumentierten Fehlercodes im Klartext und fragt nicht weiter, sobald das Budget aufgebraucht ist.
* **Echtzeit-Streaming:** Verbindet sich via WebSockets/MQTT direkt mit dem BMW-Server.
* **Robuste Authentifizierung:** Nutzt den offiziellen OAuth2 Device Code Flow.
* **Selbstheilung:** Erneuert Tokens, bevor sie ablaufen — geprüft alle 5 Minuten, erneuert bei 10 Minuten Restlaufzeit. Die Spanne deckt bewusst zwei Prüfdurchläufe ab, damit ein gescheiterter Refresh eine zweite Gelegenheit bekommt, solange der Token noch lebt.
* **Watchdog Reconnect:** Überwacht den Datenfluss und startet die Verbindung vollautomatisch neu, falls keine Daten mehr ankommen.
* **Sperre bei Störungen:** Meldet BMWs REST-API eine vorübergehende Störung, pausieren weitere Abrufe, statt sich als Fehlversuche zu häufen — 5, 15, 30, dann 60 Minuten, solange sie anhält; der erste erfolgreiche Abruf hebt sie auf. Ein Knopf **„Trotzdem abfragen“** umgeht sie für genau einen Versuch. Die Sperre liegt in der Datenbank, ein Neustart hebt sie also nicht auf. Der Live-Datenstrom über MQTT ist nicht betroffen.
* **Docker:** Verfügbar als vorgefertigtes Multi-Arch Image (amd64/arm64) über die GitHub Container Registry.
* **Strukturierte Daten:** Wandelt komplexe JSON-Objekte in saubere MQTT-Topics um (z.B. `home/bmw/live/vehicle/mileage`).
* **Multi Car Mode:** Mit `LOCAL_MQTT_APPEND_VIN=true` in der `.env` wird die VIN des jeweiligen Fahrzeugs an das Basis-Topic angehängt (z.B. `home/bmw/live/WBA…/vehicle/mileage`). So lassen sich mehrere Autos auf einem Broker sauber trennen.

### ⚠️ Danksagung & Credits

**Dieses Projekt basiert maßgeblich auf der Arbeit von:**
👉 **[whi-tw/bmw-cardata-streaming-poc](https://github.com/whi-tw/bmw-cardata-streaming-poc)**

Der Kern-Client (`lib/bmw_cardata.py`), der für die Protokoll-Implementierung und den Anmeldeprozess zuständig ist, stammt aus diesem Repository. Ein großes Dankeschön an den Autor für das Reverse-Engineering der API\!

-----

### 🚀 Installation (Docker Compose)

Der einfachste Weg ist die Nutzung des vorgefertigten Docker-Images. Es muss kein Code geklont oder manuell gebaut werden.

#### 1\. Voraussetzungen & Zugangsdaten

Um diese Brücke zu nutzen, benötigen Sie spezifische Zugangsdaten aus dem BMW Developer Portal.

  * **Client ID:** Anweisungen zur Registrierung und zum Erhalt der Client ID finden Sie hier:  
    [BMW CarData - Technische Registrierung](https://bmw-cardata.bmwgroup.com/customer/public/api-documentation/Id-Technical-registration)
  * **MQTT Username:** Ihren spezifischen MQTT-Benutzernamen finden Sie in der Streaming-Dokumentation hier:  
    [BMW CarData - Streaming Dokumentation](https://bmw-cardata.bmwgroup.com/customer/public/api-documentation/Id-Streaming)

#### 2\. Konfigurationsdateien

Erstellen Sie einen Projektordner und legen Sie folgende Dateien an:

**`.env`** (Konfiguration)
*⚠️ Sicherheitshinweis: Diese Datei niemals teilen oder auf GitHub hochladen\!*

```ini
# BMW Konfiguration
# Fügen Sie hier die Daten aus den oben verlinkten BMW-Portalen ein
CLIENT_ID=IHRE_BMW_CLIENT_ID
MQTT_USERNAME=IHR_BMW_MQTT_USERNAME

# Lokaler MQTT Broker
LOCAL_MQTT_URL=192.168.1.xxx
LOCAL_MQTT_PORT=1883
LOCAL_MQTT_USER=dein_lokaler_user
LOCAL_MQTT_PASS=dein_lokales_passwort
LOCAL_MQTT_APPEND_VIN=false #Optional für mehrauto Instanze: VIN a (z.B. home/bmw/live/WBA.../...)

# Logs
LOG_LEVEL=INFO

# Datenaufbewahrung (optional, seit v1.8.9)
# Wie lange Telemetriewerte in der SQLite-Datenbank bleiben (Standard: 30 Tage)
TELEMETRY_RETENTION_DAYS=30
# Wie lange der Standortverlauf bleibt. 0 = unbegrenzt (Standard).
# Achtung: Der Standortverlauf ist ein vollständiges Bewegungsprofil.
LOCATION_RETENTION_DAYS=0
# Abstand zwischen zwei Bereinigungsläufen in Stunden (Standard: 24)
MAINTENANCE_INTERVAL_HOURS=24

# Standortverlauf (optional)
# Seit v1.13.0 werden Breiten- und Längengrad über BMWs eigene Messzeit
# gepaart – beide Hälften müssen also aus derselben Messung stammen. 0 heißt
# exakt. Nur erhöhen, wenn das Log zeigt, dass gültige Paare verworfen werden.
GPS_MAX_MEASUREMENT_DELTA=0
# Rückfallebene für den seltenen Fall, dass BMW keine Messzeit mitliefert:
# maximaler Abstand in Sekunden zwischen dem EINTREFFEN von Breiten- und
# Längengrad (Standard: 30).
GPS_MAX_TIMESTAMP_DELTA=30

# Kartenkacheln (optional, seit v1.8.10)
# Die Kachel-Anfragen übertragen die Position deines Fahrzeugs an den hier
# eingetragenen Anbieter. Standard ist OpenStreetMap. Wer das nicht möchte,
# trägt hier einen eigenen Tile-Server ein.
# MAP_TILE_URL=http://192.168.1.50:8080/tiles/{z}/{x}/{y}.png
# MAP_TILE_ATTRIBUTION=Eigener Tile-Server
```

**`vehicle.json`** (Deine Fahrgestellnummer/VIN)

```json
{
  "vin": "WBA............."
}
```

**Konfigurations-Verzeichnisse & Dateien initialisieren** (Wichtig für Docker Volumes):

```bash
touch bmw_tokens.json   # leer lassen! Die Bridge fuellt sie bei der Erstanmeldung
mkdir -p data logs
```

#### 3\. `docker-compose.yml` erstellen

```yaml
services:
  bmw-bridge:
    image: ghcr.io/bausi2k/bmw-mqtt-bridge:latest
    
    container_name: bmw-bridge
    restart: unless-stopped
    
    ports:
      - "${WEB_PORT:-8999}:${WEB_PORT:-8999}"
    
    env_file:
      - .env
    
    volumes:
      - ./bmw_tokens.json:/app/bmw_tokens.json
      - ./vehicle.json:/app/vehicle.json
      - ./logs:/app/logs
      - ./.env:/app/.env
      - ./data:/app/data
      
    environment:
      - TZ=Europe/Vienna
```

#### 4\. Starten

```bash
docker compose up -d
```

#### 5\. Authentifizierung (Erster Start)

1.  Öffne sofort die Logs: `docker compose logs -f`
2.  Dort wird ein Link zur BMW-Webseite angezeigt.
3.  Öffne den Link im Browser und melde dich mit deiner BMW ID an, um den Zugriff zu genehmigen.
4.  Das Skript empfängt die Tokens automatisch, speichert sie in `bmw_tokens.json` und beginnt mit dem Streaming.

-----

### ⚖️ Lizenz

Dieses Projekt ist lizenziert unter **CC BY-NC-SA 4.0**.

  * ✅ **Erlaubt:** Private Nutzung, Veränderung und Weitergabe.
  * ❌ **Verboten:** Kommerzielle Nutzung oder Verkauf der Software.

Details finden Sie in der Datei [LICENSE](https://www.google.com/search?q=LICENSE).

### 🤝 Credits

**\#kiassisted** 🤖
Dieses Projekt wurde mit Unterstützung von KI erstellt.
Codearchitektur, Logik und Dokumentation wurden von Gemini unterstützt.

-----

\<a href="https://www.buymeacoffee.com/bausi2k" target="\_blank"\>\<img src="https://cdn.buymeacoffee.com/buttons/v2/default-yellow.png" alt="Buy Me A Coffee" style="height: 60px \!important;width: 217px \!important;" \>\</a\>

```
```
