# Changelog

[![Buy Me A Coffee](https://cdn.buymeacoffee.com/buttons/v2/default-yellow.png)](https://www.buymeacoffee.com/bausi2k)

Alle nennenswerten Änderungen an diesem Projekt werden in dieser Datei dokumentiert.

Das Format orientiert sich an [Keep a Changelog](https://keepachangelog.com/de/1.0.0/),
und dieses Projekt folgt [Semantic Versioning](https://semver.org/lang/de/).

> Die Historie beginnt mit v1.8.0. Ältere Einträge betreffen überwiegend interne
> Umbauten ohne Auswirkung auf den Betrieb der Bridge.

## [1.8.10] - 2026-08-07
### Fixed
- **Keine verlorenen Messwerte mehr bei gleichzeitigem Zugriff:** Jeder einzelne Telemetriewert öffnete bisher eine eigene SQLite-Verbindung — bei einer BMW-Nachricht mit Unterschlüsseln schnell ein Dutzend — und das ohne WAL-Modus und ohne Timeout. Griff die Web-UI gleichzeitig lesend zu, scheiterte der Schreibvorgang mit `database is locked`, der Messwert war verloren und wurde lediglich protokolliert. Die Datenbank läuft jetzt im WAL-Modus (gleichzeitiges Lesen und Schreiben), Verbindungen haben ein Timeout von 15 Sekunden, und alle Werte einer Nachricht werden über eine einzige Verbindung geschrieben.
- **Keine doppelten Punkte mehr im Standortverlauf:** BMW sendet dieselbe Position wiederholt; jede Wiederholung wurde als neuer Punkt gespeichert. Im Livebetrieb entstanden so drei Einträge mit identischen Koordinaten innerhalb von 21 Sekunden. Da der Standortverlauf bewusst nie automatisch bereinigt wird, blieben diese Duplikate dauerhaft liegen. Ein neuer Punkt entsteht jetzt nur noch bei tatsächlicher Bewegung (etwa ein Meter) oder bei einer Richtungsänderung ab fünf Grad.
- **Weniger verworfene Standortpunkte:** Der Plausibilitätscheck aus v1.8.3 erlaubte zwischen den Zeitstempeln von Breiten- und Längengrad nur 2,0 Sekunden Abstand. Im Livebetrieb trafen die Werte mit **20,8 Sekunden** Abstand ein und wurden dadurch verworfen. Die Grenze liegt jetzt bei 30 Sekunden und ist über `GPS_MAX_TIMESTAMP_DELTA` einstellbar; der Schutz gegen die Kombination veralteter Teilwerte bleibt erhalten.
- **Sonderzeichen zerlegen die Oberfläche nicht mehr:** Log-Zeilen und Konfigurationswerte wurden ohne Maskierung per `innerHTML` eingesetzt. Ein Anführungszeichen im MQTT-Passwort zerlegte das Konfigurationsformular, spitze Klammern in einer Log-Zeile das Log-Fenster. Beide Stellen laufen jetzt über eine gemeinsame `escapeHtml()`-Funktion.
- **Oberfläche nach einem Update sofort aktuell:** Browser lieferten `app.js` und `style.css` nach einer neuen Version weiterhin aus ihrem Cache — die Änderungen wurden erst nach einem harten Neuladen sichtbar. Der Server sendet für diese Dateien nun `Cache-Control: no-cache`, wodurch der Browser per ETag gegenprüft. Unveränderte Dateien kosten dadurch nur eine 304-Antwort; die mitgelieferten Fremdbibliotheken unter `/vendor` bleiben regulär cachebar.

### Changed
- **Die Oberfläche lädt vollständig ohne Internet:** Leaflet (147 KB) wird jetzt aus `lib/web/vendor/` ausgeliefert statt von `unpkg.com`, die Web-Schriften entfallen zugunsten eines Stacks aus Systemschriften, und das Ersatzbild des Fahrzeugs ist ein eingebettetes SVG statt eines Bildes von `cdn.pixabay.com`. In einem abgeschotteten Netz — dem erklärten Einsatzzweck der Bridge — blieb die Karte bisher leer. Von allen externen Aufrufen verbleiben nur noch die Kartenkacheln selbst.
- **Kartenquelle konfigurierbar:** Die Kachel-Anfragen enthalten die Position des Fahrzeugs und gingen fest an OpenStreetMap. Über `MAP_TILE_URL` und `MAP_TILE_ATTRIBUTION` lässt sich stattdessen ein eigener Tile-Server eintragen; ist einer gesetzt, wird OpenStreetMap nicht mehr kontaktiert. Die Konfiguration wird vor dem ersten Kartenaufbau geladen, sodass auch keine einzelne Anfrage an den Standardanbieter mehr entsteht.

### Added
- **Neues Modul `lib/location_tracker.py`:** Die Standort-Historisierung lag bisher inmitten von `on_bmw_message()` und arbeitete auf globalen Variablen — dadurch war sie weder testbar noch gegen überlappende Zugriffe abgesichert. Sie ist jetzt gekapselt, über ein Lock geschützt und mit 13 Tests abgedeckt.
- **32 weitere automatisierte Tests** für Datenbank-Nebenläufigkeit, Standort-Historisierung und Frontend-Abhängigkeiten. Alle liefen vor der Behebung rot.

## [1.8.9] - 2026-08-06
### Fixed
- **Verständliche Meldung statt Absturz bei unvollständiger Konfiguration:** Fehlte ein Pflichteintrag in der `.env`, scheiterte der Start mit einem nackten `TypeError` aus `int(None)` — noch bevor das Logging eingerichtet war, also ohne jeden Hinweis auf die Ursache. Das trat typischerweise nach dem Speichern über den Konfigurations-Editor auf. Pflichtangaben werden jetzt über das neue Modul `lib/config.py` geprüft und melden im Klartext, welcher Eintrag fehlt oder ein falsches Format hat. Gleiches gilt für eine fehlende oder unlesbare `vehicle.json`. Beide Fälle beenden die Bridge nun mit Exit-Code 1 statt 0, sodass Docker und systemd den Fehlstart als solchen erkennen.
- **Kein blockierender Anmeldedialog mehr im Dauerbetrieb:** Schlug die Token-Erneuerung fehl (etwa bei einem Netzwerkausfall), fiel `authenticate()` in den interaktiven OAuth2 Device Code Flow: Ausgabe eines Codes auf der Konsole, Öffnen eines Browsers und anschließendes Warten bis zum Ablauf. Im Container bedient das niemand, der aufrufende Thread blockierte also minutenlang. Da Token-Refresh- und Watchdog-Thread beim Beenden per `join()` erwartet wurden, blockierte das zusätzlich das Herunterfahren, bis Docker nach zehn Sekunden hart abbrach. `authenticate()`, `connect_mqtt()` und `_ensure_valid_tokens()` kennen jetzt einen Parameter `interactive`; die Hintergrund-Threads nutzen `interactive=False` und versuchen es beim nächsten Durchlauf erneut. Die Erstanmeldung auf der Kommandozeile bleibt unverändert.
- **Konfigurations-Editor zerstört die `.env` nicht mehr:** `POST /api/config` schrieb die Datei komplett neu und wickelte jeden Wert in einfache Anführungszeichen. Dabei gingen alle Kommentare verloren, und ein Apostroph im MQTT-Passwort beendete den String vorzeitig und machte die Datei unlesbar — zusammen mit dem obigen Startproblem startete die Bridge danach nicht mehr. Das neue Modul `lib/env_file.py` ändert gezielt einzelne Zeilen, erhält Kommentare, Leerzeilen und Reihenfolge, lässt nicht übermittelte Einträge unangetastet und schreibt über eine temporäre Datei, damit ein Abbruch die `.env` nicht zerstört.
- **Datenbankbereinigung läuft wieder:** `prune_telemetry()` wurde nur einmal beim Start aufgerufen. Bei `restart: unless-stopped` läuft der Container jedoch monatelang durch, sodass die 30-Tage-Grenze faktisch nie griff — im Livebetrieb waren die ältesten Einträge 47 Tage alt. Ein neuer Wartungs-Thread führt die Bereinigung jetzt standardmäßig alle 24 Stunden aus.

### Changed
- **Cache-Zeit der Reifendaten an das BMW-Tageslimit angepasst:** Die CarData-API erlaubt laut Integration Guide (v1.6, Kapitel 3.3) nur **50 Abrufe pro Tag** und antwortet danach mit `HTTP 403` und `exveErrorId: CU-429` — was leicht als Berechtigungsproblem fehlgedeutet wird. Bei einem Cache von 5 Minuten ergab ein dauerhaft geöffneter SmartMaintenance-Tab 288 Abrufe pro Tag. `TTL_TYRES` steht daher jetzt auf 3600 Sekunden (höchstens 24 Abrufe pro Tag). Reifendaten aktualisieren sich dadurch stündlich statt alle fünf Minuten; der Endpunkt unterstützt weiterhin `?bypass_cache=true` für eine sofortige Aktualisierung, im Frontend gibt es dafür bislang aber keinen Button.
- **Robusteres Herunterfahren:** Die Hintergrund-Threads laufen als Daemon-Threads und werden beim Beenden mit einem Zeitlimit von fünf Sekunden erwartet. Ein hängender Thread kann das Beenden damit nicht mehr verhindern.

### Added
- **Konfigurierbare Datenaufbewahrung:** `TELEMETRY_RETENTION_DAYS` (Standard 30), `LOCATION_RETENTION_DAYS` (Standard 0 = unbegrenzt) und `MAINTENANCE_INTERVAL_HOURS` (Standard 24). Der Standortverlauf wird weiterhin standardmäßig nie automatisch gelöscht — er stellt ein vollständiges Bewegungsprofil dar, dessen Löschung eine bewusste Entscheidung bleiben soll.
- **45 weitere automatisierte Tests** in `tests/` für Konfigurationsprüfung, `.env`-Verarbeitung, Datenbankbereinigung, nicht-interaktive Authentifizierung und die Einhaltung des API-Tagesbudgets. Alle liefen vor der Behebung rot.

## [1.8.8] - 2026-08-05
### Security
- **Härtung des Container-Images:** Das veröffentlichte Image enthält ab sofort ausschließlich den Anwendungscode. Zuvor kopierte der Build-Prozess mangels `.dockerignore` den gesamten Projektordner in die Image-Layer, wodurch Entwicklungs- und Zustandsdateien mitverpackt wurden. **Empfehlung: auf das aktuelle Image aktualisieren** — `docker compose pull && docker compose up -d`.
- **Absicherung gegen versehentlich eingecheckte Zugangsdaten:** Die Ignore-Regeln erfassen jetzt auch Sicherungskopien der Token-Datei (`bmw_tokens*.json`) sowie Datenbankdateien (`*.db`, `*.sqlite`). Der Standortverlauf in `data/` enthält ein vollständiges Bewegungsprofil des Fahrzeugs und gehört unter keinen Umständen in ein Repository. Automatisierte Tests in der CI prüfen beides ab jetzt bei jeder Änderung.

### Changed
- **Deutlich schnellere Builds:** Durch den Ausschluss von Laufzeit- und Entwicklungsdateien schrumpft der Docker-Build-Kontext von rund 124 MB auf 300 KB, was sich besonders auf dem Raspberry Pi bemerkbar macht.

### Fixed
- **Dokumentation:** Der Verweis auf den Multi-Car-Modus zeigte auf ein nicht öffentlich zugängliches Ticket. Die Einrichtung ist jetzt direkt in der README beschrieben.

## [1.8.7] - 2026-06-25
### Fixed
- **Korrektur Git-Tag-Kollision:** Behebung eines Versionskonflikts, da das Git-Tag `v1.8.6` bereits fälschlicherweise auf einen älteren Commit auf dem Server verwies. Dieses Release bündelt alle Neuerungen des veredelten Designs und Hell-/Dunkelmodus offiziell unter der neuen Version **`v1.8.7`**.

### Added
- **Einführung von umschaltbarem Hell-/Dunkelmodus (Light/Dark Mode):** Vollständige Unterstützung für ein nahtloses Umschalten zwischen hellem und dunklem Design direkt im Web-Interface über einen eleganten, abgerundeten Toggle-Button (`#theme-toggle`). Das gewählte Farbschema wird im `localStorage` gespeichert und bleibt über Sessions hinweg erhalten. Zudem wird das bevorzugte System-Farbschema des Betriebssystems (`prefers-color-scheme`) beim ersten Laden automatisch erkannt.
- **Flicker-Schutz beim Laden (IIFE):** Einbau einer sofort ausgeführten JavaScript-Funktion im Header der Applikation, um das Theme vor dem Laden des Dokuments anzuwenden. Dies eliminiert jegliches unschöne Flackern beim Laden der Benutzeroberfläche.
- **Dynamische Leaflet-Karten-Filterung:** Implementierung einer automatischen CSS-Invertierung und Farbtondrehung für Leaflet-Kartenkacheln (OpenStreetMap) im Dark-Mode. Beim Theme-Wechsel werden alle aktiven Karten (Übersichtskarte, Routenverlaufskarte und Sektorenkarten) automatisch per `layer.redraw()` neu gerendert, wodurch das Theme-Umschalten fließend und ohne teure Tile-APIs von Drittanbietern erfolgt.

### Changed
- **CGDESIGN Veredelung (v1.0.0):** Anpassung des Farbschemas nach den modernisierten Spezifikationen mit klassischen, eleganten BMW-blauen Akzentfarben (Hue `215` / `#1C69D4`) anstelle des bisherigen Violetts.
- **Weichere Ecken und Premium-Schatten:** Erhöhung und Verfeinerung der Radien-Vorgaben nach CGDESIGN v1.0.0 (`--border-radius-lg: 16px`, `--border-radius-md: 12px`, `--border-radius-sm: 8px`) für ein luxuriöseres und weicheres haptisches Erscheinungsbild aller Panels, Kacheln und Eingabefelder. Hinzufügen von subtilen Glassmorphism-Glows und fein abgestimmten Schatten-Effekten.

## [1.8.5] - 2026-06-24
### Removed
- **Schnittstelle für manuellen REST-Telemetrie-Abruf entfernt:** Bereinigung des Dashboards und des FastAPI-Backends durch Entfernen des Buttons "🔄 Manuell abholen" sowie des unbrauchbaren REST-Endpunkts `/api/pull-telemetry`. Da BMW für private B2C-Accounts keine automatischen Freigaben oder REST-Abrufe über die ExVe-Schnittstelle erlaubt, schlugen diese Versuche stets mit einem `403 Forbidden` im Docker-Log fehl. Die Fahrzeugdaten kommen stattdessen vollautomatisch und fehlerfrei über den Live-MQTT-Stream (`main.py`) an.

## [1.8.4] - 2026-06-22
### Fixed
- **Synchronisierte GPS-Aufzeichnung (Diagonalen statt Treppenstufen):** Behebung eines Darstellungsfehlers ("Treppenstufen / Rechtwinkelige Fahrtlinien"), der durch asynchron ankommende Koordinaten-Einzelwerte über den MQTT-Stream entstand. Die Positions-Historisierung puffert jetzt Breitengrad- und Längengrad-Updates separat und speichert einen neuen Track-Punkt in der SQLite-Datenbank erst dann ab, wenn *beide* Komponenten seit dem letzten Eintrag aktualisiert wurden. Dies stellt sicher, dass der Fahrtverlauf auf der Karte flüssig und mit präzisen Diagonalen entlang der echten Straßen gezeichnet wird.

## [1.8.3] - 2026-06-22
### Fixed
- **Plausibilitäts-Check bei GPS-Historisierung:** Behebung eines Darstellungsfehlers ("Viereck in der Location History"), der durch asynchron ankommende `latitude`- und `longitude`-Werte über den MQTT-Stream entstehen konnte (z.B. nach Verbindungsunterbrechungen). Es wird nun sichergestellt, dass die Zeitstempel der beiden Koordinaten maximal 2,0 Sekunden voneinander abweichen, bevor sie im Standortverlauf abgespeichert werden.

## [1.8.2] - 2026-06-21
### Added
- **Mobiles Burger-Menü:** Auf mobilen Geräten (< 768px) verwandelt sich die klassische, horizontale Tab-Navigation ab sofort in ein elegantes, vertikal ausklappbares Burger-Menü. Der Button rechts im Header animiert fließend in ein "X" bei geöffnetem Zustand. Das Menü schließt sich nach Auswahl eines Tabs vollautomatisch.

### Changed
- **Behebung von mobilem Breiten-Fehler (iPhone):** Das dreispaltige Spezifikations-Grid (`.car-spec-grid`) unter dem Fahrzeugbild wird auf Mobilgeräten nun einspaltig dargestellt, und die lange VIN wird per `word-break: break-all;` vor dem horizontalen Überlauf geschützt.
- **Reduziertes Mobile-Padding:** Das Innen-Padding von Panel-Elementen (`.glass-panel` und `.image-wrapper`) wurde auf Mobilgeräten von 32px auf 20px bzw. 16px optimiert, wodurch die Seite auf iPhones perfekt reinpasst und flüssiges vertikales Scrollen ermöglicht wird.
- **Responsive Header & Filter-Layouts:** Der App-Header und die Filter-Elemente im Standortverlauf brechen auf kleinen Bildschirmen sauber vertikal um, um gegenseitiges Quetschen der UI-Elemente zu verhindern.

## [1.8.1] - 2026-06-19
### Added
- **Automatische Verknüpfung und Consent-Anforderungsaufruf (Clearance Request):** Um den Freigabeprozess vollständig zu automatisieren, sendet die Bridge nach dem erfolgreichen Anlegen des Containers nun vollautomatisch eine Clearance-Anfrage (`POST /exve/containers/{containerId}/vehicles/{vin}`) an das Fahrzeug. Dadurch erscheint die Freigabe-Aufforderung ab sofort direkt und ohne Verzögerung im ConnectedDrive- / MyBMW-Portal des Endnutzers zur Bestätigung.
- **Zustands-Wiederherstellung aus SQLite beim Start:** Beim Booten der Bridge wird der gesamte Dashboard-Status (inklusive letzter Werte und Sparkline-Historien) sofort aus der lokalen SQLite-Datenbank geladen. Das Dashboard ist dadurch nach einem Neustart der Bridge direkt vollständig befüllt und einsatzbereit.

### Changed
- **Minimaler VSS-Fallback-Key für Containererstellung (Stufe 3):** Umstellung des absolut minimalen Ersatzschlüssels von `vehicle.vehicle.travelledDistance` auf `vehicle.drivetrain.batteryManagement.header`. Dies dient als robustes, minimales Rückgrat bei der automatischen Containererstellung, falls umfassendere Datensätze von BMW abgelehnt werden.
- **Graceful B2C-Fallback bei API-Pull-Fehlern:** Wenn ein privates B2C-Konto verwendet wird und die REST-API zur Containererstellung oder Freigabe mit `403 Forbidden` blockiert wird, wird dies nun elegant abgefangen. Das Dashboard zeigt eine informative Hinweismeldung, dass die Bridge im reinen Streaming-Modus über den Live-MQTT-Stream läuft (der bereits im Portal "ready" is), statt den Button rot zu sperren.

## [1.8.0] - 2026-06-19
### Added
- **Automatische CarData-Containererstellung (Fallback):** Sobald beim Laden der Telemetriedaten der BMW-Fehler `"No active container found for user"` auftritt, erstellt das Backend vollautomatisch einen neuen, vorkonfigurierten CarData-Container namens `"BMW-MQTT-Bridge-Container"` mit allen wesentlichen Keys (`mileage`, `gps`, `heading`, `doors`, `windows` etc.). Der Nutzer erhält eine informative `403 Forbidden` Meldung mit der Aufforderung, den neuen Container im BMW Portal mit einem Klick freizugeben.
- **Geteilte Übersichtskarte (Split-Modus):** Optionale Integration einer interaktiven Leaflet.js-Navigationskarte auf der Startseite im "Vehicle Visual"-Panel. Bei Aktivierung teilt sich das Layout elegant in 2/3 Fahrzeugbild und 1/3 Echtzeit-Karte auf.
- **Datenpunkt-Zuweisung für Übersichtskarte:** Über ein Einstellungs-Zahnrad am Fahrzeugbild-Panel lässt sich die Karte konfigurieren und die für GPS (`latitude`, `longitude`, `heading`) verwendeten Telemetrie-Keys flexibel filtern und zuweisen.
- **Vektorbasiertes Richtungssymbol (Heading):** Einbindung eines eleganten, rotierenden SVG-Richtungsanzeigers auf der Übersichtskarte passend zur aktuellen Fahrtrichtung.
- **Langzeit-Standortverlauf und Routenhistorie:** Einführung einer permanenten, nicht automatisch bereinigten SQLite-Tabelle `location_history` zur lückenlosen Archivierung aller GPS-Positionen im Hintergrund.
- **Routenhistorie-Tab (Dashboard):** Ein neuer dedizierter Reiter "Standortverlauf" visualisiert alle zurückgelegten Strecken innerhalb eines frei definierbaren Zeitraums auf einer Dark-Theme-Karte inklusive Start- und Endpunkt sowie Richtungspfeilen.
- **Mobil-Optimierungen für iPhones:** Das Split-Grid-Layout der Übersichtskarte wandelt sich auf mobilen Viewports (Breite < 768px) automatisch in eine responsive, vertikal gestapelte Ansicht um.

