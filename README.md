# BMW CarData Streaming MQTT Bridge (v1.8.9)

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
* **BMW CarData API Integration:** Fetches official vehicle image and basic details dynamically.
* **Auto-Container Fallback:** Automatically registers a pre-configured telemetry container if BMW reports `"No active container found"`.
* **Overview Split Map:** Optional Leaflet.js real-time navigation map embedded side-by-side with the car image (2/3 image, 1/3 map) with dynamic GPS and rotated vector heading markers.
* **Long-Term Route History:** Continuous SQLite logging of location coordinates with an interactive historical path map tab (time-filtered, follows the selected theme).
* **Real-time Streaming:** Connects to BMW's MQTT interface via WebSockets/MQTT.
* **Robust Authentication:** Implements the OAuth2 Device Code Flow.
* **Auto-Healing:** Automatically refreshes access tokens before they expire.
* **Watchdog Reconnect:** Monitors data traffic and automatically reconnects if the stream stalls.
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
````

**`vehicle.json`** (Your car's VIN)

```json
{
  "vin": "WBA............."
}
```

**Initialize configuration directories & files** (Required for Docker volumes):

```bash
echo "{}" > bmw_tokens.json
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
* **BMW CarData API Integration:** Lädt automatisch das offizielle Fahrzeugbild und die Fahrzeugdetails.
* **Automatischer Container-Fallback:** Registriert bei einem `"No active container found"` Fehler von BMW automatisch einen Telemetrie-Datencontainer im Nutzer-Account, sodass dieser im Portal nur noch freigegeben werden muss.
* **Geteilte Übersichtskarte:** Bindet optional eine Leaflet.js-Echtzeitkarte direkt neben dem Fahrzeugbild ein (2/3 Bild, 1/3 Karte) mit dynamischer GPS- und rotierter SVG-Richtungsanzeige.
* **Langzeit-Routenhistorie:** Protokolliert alle GPS-Koordinaten in einer permanenten SQLite-Tabelle und visualisiert die Fahrtwege in einem interaktiven Kartentab ("Standortverlauf") mit Zeitraum-Filter, passend zum gewählten Farbschema.
* **Echtzeit-Streaming:** Verbindet sich via WebSockets/MQTT direkt mit dem BMW-Server.
* **Robuste Authentifizierung:** Nutzt den offiziellen OAuth2 Device Code Flow.
* **Selbstheilung:** Erneuert Tokens automatisch im Hintergrund, bevor sie ablaufen.
* **Watchdog Reconnect:** Überwacht den Datenfluss und startet die Verbindung vollautomatisch neu, falls keine Daten mehr ankommen.
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
```

**`vehicle.json`** (Deine Fahrgestellnummer/VIN)

```json
{
  "vin": "WBA............."
}
```

**Konfigurations-Verzeichnisse & Dateien initialisieren** (Wichtig für Docker Volumes):

```bash
echo "{}" > bmw_tokens.json
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
