# Changelog

[![Buy Me A Coffee](https://cdn.buymeacoffee.com/buttons/v2/default-yellow.png)](https://www.buymeacoffee.com/bausi2k)

Alle nennenswerten Änderungen an diesem Projekt werden in dieser Datei dokumentiert.

Das Format orientiert sich an [Keep a Changelog](https://keepachangelog.com/de/1.0.0/),
und dieses Projekt folgt [Semantic Versioning](https://semver.org/lang/de/).

> Die Historie beginnt mit v1.8.0. Ältere Einträge betreffen überwiegend interne
> Umbauten ohne Auswirkung auf den Betrieb der Bridge.

## [1.24.0] - 2026-08-17
### Fixed
- **Die Tabelle zeigte nur die Straße, obwohl die volle Adresse vorlag.** Gemeldet an „Marktplatz 2", während im Export „Marktplatz 2, 1234 Musterort" steht. In der Detailzeile standen Ortsname und Adresse zusätzlich hintereinander — zweimal fast dasselbe. An 205 Vorgängen aus vier Monatsexporten gemessen, gibt es drei Muster, und nur **6 von 205** fallen unter das einfachste:

  | Name | Adresse | Anzeige |
  |---|---|---|
  | `Marktplatz 2` | `Marktplatz 2, 1234 Musterort` | Adresse |
  | `Zuhause - Ahornweg 5` | `Ahornweg 5, 1234 Musterort` | `Zuhause - Ahornweg 5, 1234 Musterort` |
  | `HPC/DC Stromanbieter GmbH` | `Gewerbering 1234 Musterort` | beide, getrennt durch `·` |

  Beim zweiten Muster steckt die Straße im Namen, aber der Ort fehlt — er wird **angehängt statt ersetzt**, denn „Zuhause" ist die nützlichere Beschriftung. Beim dritten sind Betreiber und Anschrift verschiedene Dinge und bleiben beide stehen.

### Added
- **Importierte Ladevorgänge haben jetzt eine Ladeleistung.** Energiemenge und Ladedauer stehen im Export, die Spalte blieb aber leer.
- **Ein `*` kennzeichnet Werte aus dem Export**, statt „aus dem App-Export (gerundet)" hinter jeder Zeile. Die Erklärung steht einmal als Legende unter der Tabelle — und nur dann, wenn tatsächlich gekennzeichnete Werte in der Ansicht stehen.

### Note
**Die abgeleitete Leistung ist nicht die Ladeleistung.** Sie ist Energie geteilt durch die *gesamte* Steckzeit — genau die Größe, die das Projekt in v1.11.1 bewusst verlassen hat, weil sie zusammenbricht, sobald das Auto voll am Kabel steht: gemessen 12,77 kWh in 238 Minuten, also 3,22 kW an einer 11-kW-Wallbox. Für importierte Vorgänge gibt es nichts Besseres, der Export kennt keine Ladeblöcke. Deshalb wird der Wert gezeigt, aber gekennzeichnet.

Er fließt **weder in die AC/DC-Einordnung** — 3,22 kW würden eine DC-Ladung zu AC machen — **noch in den Leistungsregler**, der weiterhin die Spitzenleistung filtert. Zwei verschiedene Größen in einem Filter wären eine stille Lüge. Ladungen ohne Spitzenleistung bleiben dort sichtbar wie seit v1.22.0.

## [1.23.0] - 2026-08-17
### Fixed
- **Der Watchdog schlug jede Nacht Fehlalarm.** Am Produktivsystem gemessen (16./17.08.2026, 24 Stunden): **fünf Warnungen**, keine davon mit einem Fehler dahinter. Er kann „Auto schläft" nicht von „Stream tot" unterscheiden — bei einem geparkten Fahrzeug bringt der Reconnect naturgemäß nichts, der Zeitstempel bleibt alt, und eine Stunde später steht dasselbe wieder da. Die Abstände wachsen jetzt: **3, 6, 12, 24 Stunden**, zurückgesetzt bei der ersten Nachricht. Nur der erste Alarm einer Stille ist eine Warnung, die Wiederholungen sind Hinweise. Aus fünf Warnungen pro Nacht wird eine.
- **Der Trenn-Handler feuerte doppelt.** Am 17.08. um 08:05:33 zwei „Unexpected disconnection"-Warnungen und zwei Reconnect-Versuche in derselben Millisekunde. Ursache: `disconnect_mqtt()` rief `loop_stop()` **vor** `disconnect()` — ohne laufenden Netzwerk-Loop geht das DISCONNECT-Paket nicht mehr raus, der alte Client blieb halb offen und meldete sich später neben dem neuen. Bei 28 Neuaufbauten am Tag. Die Reihenfolge ist umgedreht, und die Referenz wird gelöscht.

### Changed
- **Die Zeile je empfangener Nachricht steht auf DEBUG statt INFO.** Sie machte 32.929 Zeilen und 5,2 MB am Tag aus — ein Sechstel des Logs, ohne etwas zu erklären, das eine Zählung nicht genauso gut sagt. An ihre Stelle tritt alle fünf Minuten eine Zusammenfassung: rund 288 Zeilen am Tag statt 32.929. Ganz ohne Lebenszeichen auf INFO wirkte die Brücke sonst tot.
- **Die GPS-Prüfzeile lief bei jeder Nachricht mit**, auch bei Reifendruck und Türstatus: 32.783 Zeilen am Tag, davon 21.571 mit „lat_updated=False, lon_updated=False" — also „hier war gar kein Standort dabei". Sie meldet sich jetzt nur noch, wenn die Nachricht überhaupt einen Standort trug.

### Note
**Der größte Hebel braucht keine Zeile Code.** So verteilten sich die 32 MB eines Tages:

| Posten | Zeilen | MB | Anteil |
|---|---|---|---|
| DEBUG: MQTT-Weitergabe | 110.376 | 20,9 | 65 % |
| DEBUG: GPS-Prüfung | 32.783 | 5,4 | 17 % |
| INFO: Nachricht empfangen | 32.929 | 5,2 | 16 % |
| Rest | 1.492 | 0,6 | 2 % |

`LOG_LEVEL=INFO` streicht davon 82 % — von 32 MB auf 5,7 MB am Tag. Der Hinweis steht jetzt in `example.env` und im README.

**Korrektur zu v1.17.2:** Dort stand, BMW habe „kein einziges Mal von sich aus getrennt", und darauf war die Überlegung gebaut, bei einem Token-Refresh gar nicht neu zu verbinden. Das Protokoll widerlegt die Prämisse: Um 08:05:33 trennte BMW mit „Keep alive timeout". Die Neuverbindung gelang in 1,1 Sekunden — und zwar **mit dem vorhandenen Token** (54 Minuten Restlaufzeit). Ein Reconnect braucht also keinen frischen Token; ob eine *bestehende* Sitzung den Tokenablauf übersteht, bleibt offen. Der Gedanke ist weiter möglich, die damalige Begründung trägt ihn nicht.

Den Watchdog abzuschalten wäre falsch: Der Zombie-Fall — Verbindung steht, Broker hält uns für angemeldet, nichts fließt — ist real, und dass der Strom wirklich abreißen kann, zeigt derselbe Tag.

## [1.22.0] - 2026-08-17
### Added
- **Freier Zeitraum mit Datum und Uhrzeit** statt der vier festen Knöpfe. Mit dem Archiv aus v1.19.0 und dem Import aus v1.21.0 liegen 200 und mehr Ladevorgänge über Monate vor — vier Stufen reichten dafür nicht mehr. Vorbelegt sind die letzten 30 Tage, damit der Alltagsfall keine zwei Eingaben kostet.
- **Filter nach Ladeart:** Alle, nur AC, nur DC, oder ohne Angabe. Letzteres ist eine eigene Auswahl, weil „BMW sagt nichts darüber" etwas anderes ist als „weder AC noch DC".
- **Zwei Regler für die Ladeleistung**, Mindest- und Höchstwert der Spitzenleistung. Sie können sich nicht überkreuzen — sonst zeigte die Tabelle nichts an, und niemand sähe warum.
- **Ein Knopf setzt alle Filter zurück.** Bei fünf Bedienelementen muss ein Weg zurück sichtbar sein.

### Changed
- Die Zeitraum-Knöpfe (7/14/30/45 Tage) sind **entfallen** — ersetzt, nicht ergänzt. Die zuletzt gewählte Filterwahl bleibt im Browser erhalten.

### Note
**Kein Filter kostet einen BMW-Abruf.** Gefiltert wird lokal aus dem Archiv. Reicht der gewählte Zeitraum weiter zurück, als der Server zuletzt geliefert hat, wird nachgeladen — auch das liest nur die Datenbank.

**Ladungen ohne Messwert bleiben sichtbar, auch bei gesetztem Mindestwert.** Eine Sitzung ohne Ladeblöcke hat keine Spitzenleistung; sie auszublenden, sobald jemand eine Untergrenze setzt, wäre ein zweiter 0-kWh-Filter — einer, den niemand sieht und niemand ausschalten kann. Ausblenden bleibt Sache des vorhandenen Umschalters. Im Browser gemessen: „ab 50 kW" zeigt bei vier Beispielsitzungen drei — die eine schnelle Ladung plus die zwei ohne Messung.

Gefiltert wird die **Spitzenleistung**, nicht die tatsächliche Ladeleistung. Sie ist auch die Grundlage der AC/DC-Einordnung, beide Filter sprechen damit über dieselbe Größe.

Wie seit v1.17.0 rechnet die Zusammenfassung aus den **sichtbaren** Sitzungen. Ein Filter verschiebt also Anzahl, Dauer, Durchschnitt und die AC/DC-Zähler mit.

Geprüft bei 1280 und 375 Pixel; mobil stehen zwei Felder je Zeile, die beiden Regler jeweils allein.

## [1.21.0] - 2026-08-17
### Added
- **Der xlsx-Export der BMW-App lässt sich einlesen.** Knopf „📥 Export einlesen" im Ladeverlauf, danach ein Bericht: wie viele Vorgänge gelesen wurden, wie viele vorhandene Messungen ergänzt wurden und wie viele neu sind. **Kostet keinen BMW-Abruf.**
- **Fünf Felder, die die CarData-API nicht kennt:** Ladekosten, Stromtarif, Ladestandort (ein sprechender Name statt Koordinaten), Adresse und Label. Sie stehen in der Detailzeile; der Standortname ersetzt in der Tabelle die Ortsangabe, wenn er vorliegt.
- **Das Archiv lässt sich rückwärts über BMWs 45-Tage-Grenze hinaus aufbauen.** Alles Ältere liefert die API nie wieder — Monatsexporte sind der einzige Weg dorthin. An vier Exporten des Projektinhabers geprüft (April bis Juli 2026): **205 Ladevorgänge, 3181 kWh, 711,59 €**, alle gelesen.
- Gelesen wird mit der Standardbibliothek. Eine xlsx ist ein ZIP mit XML darin; `openpyxl` wäre eine weitere Abhängigkeit für eine Funktion, die gelegentlich eine Datei liest. Die Datei kommt als roher Anfragekörper statt als Formular-Upload — das spart `python-multipart`.

### Note
**Ein Importwert überschreibt nie einen API-Wert.** BMW nennt die Zahlen des Exports in der Fußnote der Datei selbst „Prognosen, die von dem tatsächlichen Ladevorgang … abweichen können": Der Export liefert `~ 12 kWh`, die API `22.979991912841797` — dieselbe Ladung. Energie, Ladezustand und Zeiten bleiben, wie die API sie gemessen hat; aus dem Export kommt nur dazu, was dort fehlt.

Vorgänge, die es **nur** im Export gibt, tragen das Merkmal `estimated`, und die Oberfläche sagt, wenn gerundete Werte in einer Summe stecken. Sie bekommen **keine AC/DC-Einordnung**: Ohne Ladeblöcke gibt es keine Spitzenleistung, und der Export sagt über die Ladeart nichts. Eine erfundene Einordnung wäre schlimmer als gar keine.

**Zwei Ladungen können in derselben Minute beginnen.** In den vier Exporten zweimal vorgekommen — je ein Fehlversuch mit 0 kWh und die echte Ladung danach, beide mit derselben Startzeit, weil der Export nur Minutengenauigkeit hat. Der Schlüssel enthält deshalb auch die Endzeit, und beim Zusammenführen wird jede API-Messung höchstens einmal vergeben. Ohne beides wäre je eine der zwei Ladungen verschwunden.

`< 1 kWh` bekommt bewusst keine Zahl — der Wert liegt irgendwo zwischen null und eins, und eine erfundene Zahl in einer Summe ist schlechter als eine Lücke. Der Originaltext bleibt daneben stehen.

**Die Spalte „Entladene Strommenge" ist in allen vier Monaten leer** — 205 Vorgänge, kein einziger Wert. Damit ist die offene ROADMAP-Frage zu `energyDischargedKwh` beantwortet: BMW füllt das Feld weder in der API noch im Export.

## [1.20.0] - 2026-08-14
### Fixed
- **Der Hell-Modus legte 40 % Schwarz über die Detaildaten.** Gemeldet als „das Grau der Kacheln ist viel zu intensiv", im Browser nachgemessen: Die Seite ist `rgb(244,245,248)`, die Abschnittsfläche stand fest auf `rgba(18,18,18,0.4)` — macht **`rgb(154,154,156)`**. Die Kacheln darauf sind zu 75 % weiß und durchscheinend und wirkten dadurch grau (`rgb(230,230,230)`). **Der Fehler saß nicht in der Kachel, sondern unter ihr** — eine Korrektur an der Kachelfarbe hätte nichts gebracht. Jetzt hebt sich die Fläche im Hell-Modus nach oben ab: rund `rgb(250,250,252)`, eine Andeutung statt eines Kastens.
- **Zwölf weitere fest verdrahtete Farben laufen jetzt über Variablen.** Sie stammten alle aus der Fassung für den Dunkel-Modus und waren der Grund, dass der Hell-Modus stellenweise wie ein halb umgeschalteter Dunkel-Modus aussah. Ausgenommen bleibt der Schleier hinter Dialogen — der ist in beiden Modi dunkel. Ein Test verbietet neue feste Flächenfarben.

### Added
- **Die Kacheln lassen sich sortieren:** eigene Reihenfolge, A–Z oder zuletzt aktualisiert. Die Auswahl sitzt in der Abschnittskopfzeile und bleibt im Browser erhalten.

### Note
Sortiert wird nur die **Anzeige**. Die per Drag and Drop gewählte Reihenfolge liegt in `uiConfig` auf dem Server und bleibt unangetastet — sie ist weiterhin die Voreinstellung.

Der heikle Teil daran: `data-index` am Widget ist der Platz in `section.cards` und steuert Löschen und Bearbeiten. Wanderte er beim Sortieren mit der Anzeige, löschte ein Klick auf den Papierkorb die falsche Kachel. Die Sortierung reicht deshalb den ursprünglichen Index durch, und ein Test prüft für jede Kachel, dass er auf sie zeigt.

Aus demselben Grund ist **Ziehen bei aktiver Sortierung abgeschaltet**: Ein Zug hätte das Ziel aus der Anzeige-Position berechnet und die falschen Kacheln in `uiConfig` vertauscht.

Kacheln ohne Zeitstempel stehen bei „zuletzt aktualisiert" am Ende — nie aktualisiert ist nicht dasselbe wie gerade eben.

Geprüft bei 1280 und 375 Pixel, in beiden Modi. Im Dunkel-Modus ist der gemessene Wert byteweise derselbe wie zuvor.

## [1.19.0] - 2026-08-14
### Added
- **Ladevorgänge liegen jetzt dauerhaft in SQLite.** Bis v1.18.1 wurde der gesamte Verlauf bei jedem Abruf neu von BMW geholt — acht bis zehn Seiten, jede ein Abruf aus einem Tagesbudget von 50. Geholt wird nur noch, was neu dazugekommen ist: **im Normalfall eine Seite.** Der Vormittag des 14.08.2026 hätte damit statt 29 Anfragen etwa drei gekostet.
- **Das Archiv wächst über BMWs 45-Tage-Grenze hinaus.** Ältere Zeiträume lehnt die API mit CU-401 ab; was einmal gespeichert ist, bleibt aber. Nach einigen Monaten enthält das Archiv mehr, als BMW überhaupt ausliefern kann. Die Ansicht ist deshalb **nicht mehr auf 45 Tage begrenzt** — `days=90` lieferte bisher HTTP 400.
- **Ein abgebrochener Durchlauf ist fortsetzbar.** Jede Seite wird sofort abgelegt, nicht erst am Ende. Bricht BMW auf Seite acht ab, stehen die sieben davor im Archiv, und Aktualisieren sammelt weiter.

### Changed
- **Das Delta-Fenster überlappt um drei Tage** (`CHARGING_OVERLAP_DAYS`). Eine Ladung, die beim Abruf noch lief, steht unvollständig in der Datenbank — ohne Endzeit und mit zu wenig Energie. Ohne Überlappung bliebe dieser erste, falsche Stand für immer stehen; jetzt wird sie beim nächsten Abruf überschrieben.
- **Gespeichert wird BMWs Rohantwort.** Die AC/DC-Einordnung und die Leistungsberechnung sind Ableitungen von uns. Lägen sie eingefroren in der Datenbank, erreichte eine spätere Korrektur die alten Zeilen nicht mehr — dieselbe Überlegung wie beim Steckbrief in v1.16.0. Ein Test hebt die AC/DC-Schwelle an und prüft, dass sich die Einordnung gespeicherter Zeilen mitbewegt.
- **Der Ladeverlauf liegt nicht mehr im Zwischenspeicher von v1.18.0.** Zwei Quellen für dieselben Daten wären ein Fehler. Stammdaten, Bild, Reifen und Fahrzeugliste bleiben dort.
- **`pages` zählt jetzt die Seiten, die dieser Aufruf geholt hat** — null heißt, alles kam aus dem Archiv. **`incomplete` beschreibt das Archiv**, nicht mehr die Ansicht: Angezeigt wird immer alles Gespeicherte, offen ist nur, ob BMW noch ältere Vorgänge schuldet. Neu ist `archived`, die Gesamtzahl im Archiv.
- **Ein Zeitraumwechsel kostet keinen Abruf mehr.** Bisher hatte jeder Zeitraum einen eigenen Zwischenspeicher-Schlüssel — das war am 14.08.2026 der zweite Teil der 28 Anfragen.

### Note
Schlüssel ist `(vin, start_time)`. BMW liefert keine Sitzungs-ID; ein Fahrzeug kann aber nicht zwei Ladungen in derselben Sekunde beginnen.

Ladevorgänge werden **nie** bereinigt, wie der Standortverlauf. Telemetrie wird nach 30 Tagen gelöscht; für ein Archiv wäre das sinnlos.

## [1.18.1] - 2026-08-14
### Fixed
- **Ein Fehler auf Seite acht verwarf die sieben Seiten davor.** Am Produktivsystem gemessen (14.08.2026, 07:52:08 bis 07:52:50, direkt nach dem Deploy von v1.18.0): BMW brach den Ladeverlauf mit `CU-500` ab, **zweimal reproduzierbar bei derselben Seite**. Beide Male wurden sieben bereits bezahlte Seiten weggeworfen, und der zweite Anlauf kaufte sie erneut. Bilanz: **19 Anfragen, kein einziger angezeigter Ladevorgang**, danach war das Tagesbudget von 50 für 18 Stunden aufgebraucht. Der Server liefert jetzt aus, was bis zum Abbruch kam.
- **Das Teilergebnis wird zwischengespeichert.** Sonst kostet jeder erneute Aufruf wieder alle Seiten bis zur defekten. Wer es trotzdem noch einmal versuchen will, drückt Aktualisieren — das umgeht den Zwischenspeicher wie bisher.
- **Die Oberfläche benennt den Abbruch**, statt eine gekürzte Liste als vollständig auszugeben: „BMW hat den Verlauf nach N Seiten abgebrochen." Die Antwort trennt dafür zwei Fälle, die vorher beide nur `has_more` setzten — `incomplete` heißt „BMW hat uns unterbrochen", `has_more` allein heißt „unsere eigene Seitengrenze".

### Note
**Korrektur zu v1.17.1:** Dort steht, ein `CU-500` sei ein vorübergehender Fehler. Das trifft hier nicht zu — BMW scheitert an einer festen Stelle im Verlauf, zweimal identisch. Wiederholen hilft nicht.

Damit wird auch verständlich, warum der alte Rückfall auf 30 Tage funktioniert hatte: BMWs defekter Datensatz ist älter als 30 Tage, der kürzere Zeitraum erreichte ihn nicht. Das war Zufall, kein verlässliches Verhalten — es kostete jedes Mal einen zweiten vollständigen Durchlauf und hätte bei einem jüngeren Defekt gar nichts gebracht. Die Einschränkung des Rückfalls auf HTTP 400 bleibt daher richtig; sie hat den darunterliegenden Fehler nur sichtbar gemacht.

Der Verwurf steckte **schon vor v1.18.0** im Code: Der `try`-Block umschließt die ganze Blätterschleife, und das Ergebnis entsteht erst danach.

## [1.18.0] - 2026-08-14
### Added
- **Der Zwischenspeicher übersteht einen Neustart.** Bis v1.17.2 war er ein nacktes Dict im Arbeitsspeicher — nach jedem Deploy leer, und der erste Aufruf der Oberfläche kaufte alles noch einmal. Am Produktivsystem gemessen (13.08.2026, 14:41:16 bis 14:41:32): **13 Anfragen in 16 Sekunden**, davon elf Seiten Ladeverlauf; die letzte lief in ein CU-429, das Tagesbudget war aufgebraucht. An einem Entwicklungstag mit drei, vier Deploys ist ein Budget von 50 allein damit weg. Die Antworten liegen jetzt in derselben SQLite wie Telemetrie und Standortverlauf, also im Volume `./data`.
- **Gespeichert wird der Zeitpunkt des Abrufs, nicht der des Ladens.** Sonst hätte ein Neustart die Aufbewahrungsdauer verlängert statt sie zu wahren — aus 24 Stunden würden bei täglichem Neustart beliebig viele. Ein abgelaufener Eintrag wird nach dem Start ganz normal nachgeholt.
- **Das Fahrzeugbild überlebt als Bytes**, base64-kodiert in der JSON-Ablage. Es ist die einzige Antwort, die kein JSON ist.

### Changed
- Der Wartungs-Thread räumt jetzt auch abgelaufene API-Antworten weg (älter als 48 Stunden, doppelt so lang wie die längste Aufbewahrungsdauer). Ohne das wüchse die Tabelle mit jedem je abgefragten Zeitraum.

### Note
**Verhaltensänderung:** Bisher war „Container neu starten" der stille Weg, frische Daten zu erzwingen. Das funktioniert nicht mehr — sonst wäre die ganze Übung sinnlos. Der ausdrückliche Weg bleiben die Aktualisieren-Knöpfe; sie überschreiben die abgelegte Zeile.

Eine Korrektur an der ursprünglichen Planung: Ein gesondertes Löschen der gespeicherten Zeile bei `bypass_cache` ist **nicht** nötig. Ein erfolgreicher Abruf überschreibt sie ohnehin, und scheitert er, ist der alte Wert das Beste, was noch da ist.

Die Persistenz ist eine Bequemlichkeit, keine Notwendigkeit: Fällt die Datenbank aus, arbeitet der Zwischenspeicher wie vorher im Arbeitsspeicher weiter und übersteht dann eben keinen Neustart. Ein Abruf scheitert daran nicht.

Die Falle aus v1.16.0 bleibt geschlossen und wird jetzt schärfer geprüft: Gespeichert wird ausschließlich BMWs Rohantwort. Die nutzbare Energie kommt bei jeder Auslieferung frisch aus dem Datenstrom — läge sie im Zwischenspeicher, wäre sie einen Tag lang eingefroren, und mit der Persistenz überstünde dieser eingefrorene Wert sogar den Neustart.

## [1.17.2] - 2026-08-14
### Changed
- **Der Token wird alle 5 Minuten geprüft, erneuert wird bei 10 Restminuten** (vorher 15 Minuten Takt, 20 Minuten Vorlauf). Der Datenstrom startet damit alle 50 statt alle 45 Minuten neu — 29 statt 32 Mal am Tag. Eine Prüfung ohne fälligen Refresh ist ein Datumsvergleich im Speicher: kein HTTP, kein Tagesbudget. Der enge Takt kostet also nichts.
- **Ein gescheiterter Refresh bekommt jetzt eine zweite Gelegenheit.** Das ist der eigentliche Gewinn. Die alte Einstellung erneuerte bei 15 Restminuten und sah 15 Minuten später wieder nach — da war der Token bereits tot. Es gab genau **einen** Versuch; scheiterte er, war die Verbindung weg. Die Bedingung im Test zählt deshalb jetzt Durchläufe statt Minuten und verlangt zwei.
- **Prüfintervall und Vorlauf stehen als Konstanten nebeneinander** in `lib/bmw_cardata.py`. Zwei Zahlen an zwei Orten — die eine in `main.py`, die andere in der Bibliothek — waren die Ursache der alten Fehleinstellung.

### Note
Über die Kadenz ist nicht viel mehr zu holen: Solange jede Erneuerung den MQTT-Client neu startet, liegt die Untergrenze bei 24 Neustarts am Tag, weil der Token eine Stunde lebt. Gemessen wurden 32, jetzt sind es 29, das Minimum wären 24.

Der eigentliche Hebel wäre, bei einem Refresh **gar nicht** neu zu verbinden. BMW authentifiziert die MQTT-Sitzung beim CONNECT, und in 16 Stunden Protokoll (13./14.08.2026) hat BMW von sich aus kein einziges Mal getrennt. Ob die Sitzung den Tokenablauf übersteht, lässt sich daraus aber **nicht** ableiten — wir starten immer vorher neu. Das wäre ein eigener Versuch.

## [1.17.1] - 2026-08-14
### Fixed
- **Ein Abruf des Ladeverlaufs kostete bis zu 28 BMW-Anfragen statt zehn.** Am Produktivsystem gemessen (14.08.2026, 06:51:39 bis 06:51:53): Der Inhaber holte den Ladeverlauf **einmal**, BMW sah 28 Anfragen. Vom Tagesbudget von 50 waren damit vor sieben Uhr morgens 29 verbraucht. Zwei Ursachen haben sich multipliziert:
  - **Nichts hielt einen zweiten Durchlauf auf.** Der Zwischenspeicher wird erst geschrieben, wenn die letzte Seite da ist — während der sieben Sekunden, die das dauert, verfehlte ihn jede weitere Anfrage und blätterte selbst los. Im Browser dasselbe: `chargingLoaded` wurde am *Ende* von `fetchChargingHistory` gesetzt und griff damit genau dann nicht, wenn es gebraucht wurde.
  - **Ein `CU-500` wurde als Zeitraum-Problem gedeutet.** Der Rückfall von 45 auf 30 Tage ist dafür da, dass BMW zu große Zeiträume ablehnt (HTTP 400, CU-401). Geprüft wurde aber nur `!res.ok`, also fiel auch ein vorübergehender Serverfehler darunter. Beide Durchläufe liefen in je einen 500er — aus zwei Durchläufen wurden vier.

### Changed
- **Gleichzeitige Anfragen an denselben Endpunkt teilen sich einen Abruf.** Wer als Zweiter kommt, wartet auf das Ergebnis des Ersten, statt selbst bei BMW anzufragen. Das gilt für alle fünf Endpunkte — Reifen, Stammdaten, Bild, Ladeverlauf und Fahrzeugliste —, denn der Fehler steckte in keinem von ihnen, sondern im Muster.
- **Auch zweimal „Aktualisieren" kostet nur einen Durchlauf.** `bypass_cache` hebt die Sperre nicht auf; wer wartet, bekommt aber ausschließlich, was *während* seiner Wartezeit entstanden ist. Ein alter Eintrag zählt beim ausdrücklichen Auffrischen nicht.

### Note
Die Sperre bringt eine Verhaltensänderung mit: Eine zweite Anfrage wartet jetzt, statt parallel zu laufen. Hängt BMW, kann das bei zehn Seiten à 20 Sekunden Zeitüberschreitung dauern — vorher hätte der Zweite eine eigene, ebenso langsame Antwort bekommen, nur eben für den doppelten Preis.

Der Prüfstand für diese Tests hatte im ersten Entwurf selbst einen Fehler: Zwei gleichzeitige Durchläufe teilten sich einen Seitenzähler, wodurch der wichtigste Test grün aussah, obwohl beide blätterten. Die Seitenzahl hängt jetzt am `nextToken`.

## [1.17.0] - 2026-08-13
### Added
- **Ladevorgänge ohne geladene Energie lassen sich ausblenden.** BMW führt im Verlauf auch Sitzungen, bei denen nichts im Akku ankam — teils mit 0 kWh, teils ganz ohne Wert. In der Tabelle sieht man den Unterschied nicht, und beide verwässern die Kennzahlen. Der Umschalter „0 kWh ausblenden" sitzt neben der Zeitraumwahl, filtert rein lokal und kostet damit **keinen BMW-Abruf**. Der Zustand bleibt im Browser erhalten.

### Changed
- **Der Umschalter verschiebt auch die Zusammenfassung** — sie rechnet aus den sichtbaren Sitzungen. Anzahl, Gesamtdauer und die AC/DC-Zähler sinken, die Gesamtenergie bleibt gleich (Nullen tragen nichts bei), und der **Durchschnitt steigt**: aus „Ø je Einsteckvorgang" wird „Ø je echter Ladung". Gemessen an vier Beispielsitzungen: 54 kWh bleiben 54 kWh, der Schnitt geht von 18 auf 27 kWh.
- **Der Zustand von Umschaltern hängt jetzt an `aria-pressed`.** Beim Ansehen im Browser gefunden: die Hervorhebung für gedrückte Knöpfe galt ausschließlich innerhalb von `.charging-range`. Der neue Knopf saß daneben und sah eingeschaltet genauso aus wie ausgeschaltet — der Filter hätte unbemerkt laufen können. Ein Test hält die Regel jetzt fest.

### Note
**Voreingestellt ist der Filter aus.** Stilles Weglassen hat dieses Projekt schon einmal teuer bezahlt — bis v1.15.0 endete der Ladeverlauf kommentarlos bei zehn Vorgängen. Ist der Filter an, steht die Zahl der ausgeblendeten Vorgänge über der Tabelle, neben dem Seitenhinweis.

Als leer gilt auch ein negativer Wert: Rückspeisung gehört nicht in eine Ladebilanz. 0,01 kWh dagegen bleiben stehen — wenig ist nicht nichts.

Die Verhaltenstests führen den echten Code aus `app.js` in Node aus, statt Zeichenketten zu vergleichen. Fehlt Node, werden sie übersprungen; auf `ubuntu-latest`, wo die CI läuft, ist es vorhanden.

**Nicht geklärt:** wodurch diese Einträge entstehen — abgebrochene Ladung, reine Vorklimatisierung oder Stecker ohne Freigabe. Der Hinweis zählt sie deshalb nur, statt sie zu deuten.

## [1.16.0] - 2026-08-13
### Added
- **Die nutzbare Energie der Batterie steht im Steckbrief.** Der Datenstrom liefert sie längst als `batteryManagement.maxEnergy` — bisher landete der Wert ungenutzt in der Datenbank. Gegenprobe an echten Daten, Ladestand mal Restkapazität gegen „kWh bis voll": 35 % bei 76 → 49 (gerechnet 49,4), 50 % bei 77 → 38 (38,5), 49 % bei 77 → 39 (39,3). Dreimal deckungsgleich. Der Wert kostet **keinen einzigen API-Abruf**.

### Changed
- **Die Nennkapazität trägt jetzt ihre Einheit: `210,6 Ah`.** In v1.14.0 stand sie roh da, weil BMW keine Einheit mitliefert. Sie ist inzwischen belegt: 210,6 Ah × 400 V = 84,24 kWh, und ein i5 xDrive40 hat rund 84 kWh brutto. Rückwärts gerechnet ergibt das 400,3 V — die Nennspannung des Gen5-Akkus. Zwei unabhängige Größen treffen sich auf ein Zehntel Volt.
- **Der Steckbrief wird bei jedem Abruf neu gebaut.** Gepuffert wird nur noch BMWs Rohantwort. Andernfalls wäre der Energiewert im tagesalten Zwischenspeicher eingefroren; er ändert sich stündlich.

### Note
**Aus den Amperestunden wird trotzdem keine Energie gerechnet.** Dazu bräuchte es die Nennspannung, und die liefert BMW in keinem Feld — im gesamten Projekt existiert keine. Sie fest auf 400 V zu setzen wäre bequem und falsch: die Neue Klasse fährt mit 800 V. Diese Bridge läuft auf fremden Fahrzeugen, dort zeigte dieselbe Formel die halbe Kapazität an, ohne dass es jemandem auffiele. Nötig ist die Rechnung ohnehin nicht — der gemessene Wert ist von der Systemspannung unabhängig.

Zwei Dinge, die **nicht** belegt sind: Der Wert liegt unter der Werksangabe von 81,2 kWh netto — ob das Alterung ist oder BMW etwas anderes meint, ist offen. Und er ist nicht konstant: innerhalb von neun Stunden wanderte er von 76 auf 77, vermutlich temperaturabhängig. Er erscheint deshalb als „Nutzbare Energie", nicht als Kapazität, und der Hinweis in der Oberfläche sagt das auch.

Schickt BMW zum Messwert eine eigene Einheit mit, hat sie Vorrang vor unserer Annahme `kWh`. Der Test, der „kWh" im Steckbrief verbietet, gilt unverändert für alles, was **nicht** aus dem Datenstrom stammt.

## [1.15.1] - 2026-08-13
### Changed
- **Die Log-Ansicht zeigt den jüngsten Eintrag oben.** Bisher lief sie zeitlich aufsteigend und sprang beim Aktualisieren ans Ende. Wer ins Log sieht, will aber wissen, was *gerade* passiert ist — nicht, was tausend Zeilen vorher war. Entsprechend gilt jetzt der obere Rand als mitlaufend: Steht die Ansicht oben, bleibt sie oben und zeigt die frischen Zeilen.

### Note
Die Schnittstelle `/api/logs` selbst bleibt **zeitlich aufsteigend**. Sie ist ein Datenstrom, keine Ansicht, und Auswertungen außerhalb der Oberfläche verlassen sich auf diese Reihenfolge. Umgedreht wird ausschließlich beim Zeichnen; ein Test hält beides auseinander.

## [1.15.0] - 2026-08-13
### Fixed
- **Der Ladeverlauf zeigte immer nur zehn Vorgänge — egal welcher Zeitraum gewählt war.** BMW liefert den Verlauf **seitenweise**, zehn Sitzungen je Seite, und nennt im Feld `next_token` die Fortsetzung. Seit v1.9.0 holte die Bridge bewusst nur die erste Seite, um das Tagesbudget zu schonen. Diese Entscheidung war verteidigbar — sie zu verschweigen nicht: `has_more` stand seit jeher in der Antwort und wurde nirgends angezeigt. Am Produktivsystem gemessen: 45 Tage angefragt, 5,4 Tage geliefert, kein Hinweis.

  Die Bridge blättert jetzt bis zu zehn Seiten durch (100 Ladevorgänge). Reicht das einmal nicht, steht es über der Tabelle, statt still abgeschnitten zu werden.

### Changed
- **Die Oberfläche holt immer den größten Zeitraum und filtert lokal.** 7, 14, 30 und 45 Tage kommen dadurch aus einem einzigen Abruf. Ein zweiter Zeitraum hätte alle Seiten erneut gekostet.
- **Rückfallebene, falls BMW den Zeitraum ablehnt:** Die Grenze ist undokumentiert (gemessen: 45 Tage angenommen, 60 abgelehnt). Schlägt der Abruf über 45 Tage fehl, versucht die Oberfläche einmalig 30 Tage, statt leer zu bleiben.

### Note
Das Tagesbudget steigt dadurch von 28 auf **37 der 50 erlaubten Abrufe** (Grenze mit Sicherheitsreserve: 40). Der Ladeverlauf kostet künftig bis zu zehn statt einem Abruf — einmal täglich, danach greift der Zwischenspeicher. Der mit Abstand größte Posten bleiben die Reifen mit 24 Abrufen (TTL eine Stunde, gerechnet für einen dauerhaft geöffneten Tab).

Der Budgettest kennt jetzt den Unterschied zwischen „ein Abruf" und „ein Abruf mit mehreren Seiten". Ohne das hätte er 28 statt 37 gemeldet und die Überschreitung erst im Betrieb sichtbar werden lassen. Für einen weiteren Endpunkt — etwa die geplanten Ladeorte — wird es damit eng; dort wäre zuerst die TTL der Reifen zu überdenken.

## [1.14.2] - 2026-08-13
### Fixed
- **Der Datenstrom wurde alle 15 Minuten neu aufgebaut.** Am Produktivsystem beobachtet: sechs Erneuerungen des Zugriffstokens in 75 Minuten, jede mit einem Neustart der MQTT-Verbindung — bei einem Token, der **eine Stunde** gültig ist. Kommt während der Trennung etwas von BMW, ist es verloren.

  Ursache war nicht das Prüfintervall, sondern die Reihenfolge in `authenticate()`: Die Methode ruft `_load_tokens()` auf, und das **ersetzt** `self.tokens` durch den Dateiinhalt. Gespeichert werden aber nur `refresh_token`, `gcid` und `scope` — der gültige `id_token` war danach weg, und jede Ablaufprüfung lief ins Leere. Der Refresh wurde dadurch unvermeidlich. Die Prüfung steht jetzt **vor** dem Laden; die richtige Reihenfolge stand mit `_ensure_valid_tokens()` schon im selben Modul.

- **Die Sicherheitsspanne war kleiner als das Prüfintervall.** `_is_token_expired()` rechnete mit fünf Minuten Vorlauf, der Refresh-Thread läuft alle fünfzehn. Ein Token konnte dadurch bis zu zehn Minuten abgelaufen sein, bevor überhaupt jemand nachsah — während die MQTT-Verbindung ihn weiterverwendete. Die neue Konstante `TOKEN_REFRESH_MARGIN_MINUTES` steht auf 20 Minuten und damit über dem Intervall; ein Test hält beide Werte aneinander fest, damit niemand am einen dreht, ohne das andere mitzuziehen.

### Note
Die Spanne ist bewusst **relativ zum Ablaufzeitpunkt** gewählt, nicht zur Lebensdauer. `expires_at` liefert BMW mit; ein fest verdrahtetes „nach 59 Minuten erneuern" wäre falsch, sobald BMW die Gültigkeit kürzer ansetzt.

Über acht Stunden simuliert, bei 15-Minuten-Takt und einstündiger Gültigkeit: **33 Erneuerungen vorher, 11 nachher** — 32 gegenüber 10 Neustarts des Datenstroms. In beiden Varianten war der Token zu keinem Prüfzeitpunkt abgelaufen; der neue Takt liegt bei 45 Minuten und lässt damit 15 Minuten Reserve.

Der voreingestellte Vorlauf von `_is_token_expired()` bleibt bei fünf Minuten — er gilt für Prüfungen unmittelbar vor der Verwendung, und deren Verhalten ändert sich nicht.

## [1.14.1] - 2026-08-13
### Fixed
- **Derselbe verworfene GPS-Fall stand vielfach im Protokoll.** Stimmen die Messzeiten von Breiten- und Längengrad nicht überein, kehrt die Prüfung zurück, *bevor* die Flags zurückgesetzt werden — absichtlich, denn eine später eintreffende passende Hälfte muss sich noch mit der wartenden ersten verbinden können. Folge davon ist, dass **jede** weitere BMW-Nachricht dieselbe Prüfung erneut auslöst, auch eine über Reifendruck oder Ladeleistung. Am Containerlog vom 13.08.2026 gemessen: **236 Zeilen für 13 tatsächliche Vorfälle**, einer davon 48-mal innerhalb einer Sekunde. Jeder Fall wird jetzt nur noch einmal protokolliert; unterschieden wird nach dem Paar der Messzeiten, nicht nach deren Abstand.
- An der Auswertung selbst ändert sich nichts — die Prüfung läuft weiter bei jeder Nachricht, nur die wiederholte Zeile entfällt.

### Note
Die Auswertung des Logs hat gleich zweierlei gezeigt. Erstens die Verteilung der abweichenden Messzeiten, die bisher fehlte — alle 13 Vorfälle einer 42-Minuten-Fahrt: 2, 4, 8, 13, 30, 78, 80, 84, 92, 105, 116, 181 und 181 Sekunden. Acht davon liegen über einer Minute, **im Sekundenbereich gibt es keine Häufung**. `GPS_MAX_MEASUREMENT_DELTA = 0` bleibt damit richtig; eine Toleranz von 10 Sekunden würde 3 der 13 Fälle zusätzlich zulassen, und zwar ohne Gewinn.

Zweitens: Ohne diese Korrektur wären die Zahlen bei jeder künftigen Auswertung um den Faktor 18 verzerrt gewesen. Genau das ist bei der ersten Auswertung dieses Logs passiert.

## [1.14.0] - 2026-08-12
### Added
- **Fahrzeugsteckbrief:** Der Endpunkt `basicData` wird täglich abgerufen und liefert 22 Felder. Angezeigt wurden davon **zwei** — `brand` und `modelName`, zusammengesetzt zu „BMW i5 xDrive40". Unter den Fahrzeugdaten steht jetzt ein aufklappbarer Steckbrief mit Baureihe, Karosserie, Türen, Farbe, Baudatum, Antriebsart, Motorkennung, Lademodi, Navigation, Schiebedach, Head-Unit, Lenkung, SIM-Status, Land und der vollständigen Sonderausstattungsliste. **Ohne einen einzigen zusätzlichen API-Abruf** — die Daten kamen bereits, sie wurden nur verworfen.
- **Neues Modul `lib/vehicle_profile.py`:** Übersetzt Codes in Klartext (`BEV` → Elektro, `LL` → Linkslenker, `ACTIVE` → aktiv) und formatiert das Baudatum. **Unbekannte Codes werden unverändert durchgereicht** — ein geratenes Klartextwort wäre schlechter als der Rohwert, den man nachschlagen kann. Fehlende Felder erzeugen keine Zeile statt eines Strichs.

### Note
**Die Nennkapazität ist keine Energieangabe.** `reessNominalCapacityGross` liefert `210.6`; ein i5 xDrive40 hat rund 84 kWh. Der Wert entspricht der Amperestunden-Angabe der Batterie, BMW schreibt keine Einheit dazu, und `hvsMaxEnergyAbsolute` — das Feld, das tatsächlich Energie enthielte — wird für dieses Fahrzeug nicht geliefert. Der Wert erscheint deshalb roh mit einem Hinweis; ein Test stellt sicher, dass im gesamten Steckbrief nirgends „kWh" auftaucht. Die ursprüngliche Absicht, daraus den Ladestand in kWh zu rechnen, ist damit hinfällig.

**Die Spezifikation ist weder vollständig noch verbindlich.** Sieben dort genannte Felder werden nicht geliefert (darunter `vin`, `colourCode` und `hvsMaxEnergyAbsolute`), vier ungenannte sehr wohl — unter anderem `colourDescription` („BLACK SAPPHIRE METALLIC"), das nützlicher ist als der dokumentierte Farbcode, sowie `chargingModes` und `seriesDevt`. Der Steckbrief verträgt beides.

## [1.13.0] - 2026-08-12
### Changed
- **Breite und Länge werden über BMWs Messzeit gepaart, nicht über die Ankunftszeit.** Das ist die Ursache der rechtwinkeligen Linien, die den Standortverlauf seit jeher begleiten. Der bisherige Schutz `GPS_MAX_TIMESTAMP_DELTA` (30 s) verglich, wann beide Hälften *eintrafen* — ein Längengrad, den BMW vor Minuten gemessen und eben erst geliefert hat, trägt aber die Ankunftszeit „jetzt". Der Abstand ist null, das Paar wird akzeptiert, und aus altem Breiten- und neuem Längengrad entsteht ein Knick.
- **Gespeichert wird die Messzeit.** `location_history.timestamp` enthielt bisher den Zeitpunkt, zu dem die Nachricht ankam — im Median rund 3 Sekunden nach der Messung. Künftig steht dort, wann das Fahrzeug tatsächlich an diesem Ort war. **Ältere Zeilen behalten die Ankunftszeit**, die Spalte ist also gemischt; der Unterschied liegt im Bereich weniger Sekunden.
- **Das Frischefenster entfällt**, wo eine Messzeit vorliegt. Seine Aufgabe war, dieselbe Position nicht wiederholt zu speichern — das erledigt jetzt der direkte Vergleich der Messzeit.
- **Neue Einstellung `GPS_MAX_MEASUREMENT_DELTA`** (Standard `0`, also exakt). Aus den Livedaten ist bekannt, dass 60,4 % der Paarungen exakt übereinstimmen; wie sich die übrigen verteilen, ließ sich aus Minimum, Maximum und Median nicht ablesen. Der Wert ist deshalb einstellbar statt geraten, und verworfene Paare werden mit ihrem Abstand protokolliert.
- **Rückfallebene:** Fehlt die Messzeit, gilt wieder die Prüfung auf Ankunftszeiten samt Frischefenster. Über 11.078 Nachrichten lag die Abdeckung bei 100 % — das ist eine Beobachtung, keine Zusage von BMW.

### Note
Der Beleg stammt aus der Fahrt vom 12.08.2026, drei aufeinanderfolgende Punkte:

```
07:03:31   48.425757  15.776218
07:03:49   48.425757  15.774451    dNord =    0,0 m   dOst = -130,6 m
07:03:50   48.420586  15.774451    dNord = -575,6 m   dOst =    0,0 m
```

Der Breitengrad um 07:03:49 ist zeichengenau derselbe wie 18 Sekunden zuvor, danach der Längengrad. Ergebnis: zwei achsenreine Schenkel, der zweite mit 576 Metern in einer Sekunde.

Die Sonde aus v1.11.0 hat das Ausmaß beziffert: **60,4 % der Paarungen** trugen identische Messzeiten, der größte Versatz lag bei **4.331 Sekunden** — 72 Minuten. Zwei von fünf Paarungen kombinierten also Hälften aus verschiedenen Messungen.

Eine Simulation mit diesem Anteil (400 Messungen, keine Messung, sondern eine Veranschaulichung) zeigt den Unterschied: Die alte Logik zeichnet 438 Punkte auf, davon 263 falsch gepaart — mehr Punkte als es Messungen gab, weil Nachzügler Scheinpunkte erzeugen. Die neue liefert 376 Punkte, davon **keinen falsch gepaarten**. Die Zahl der Punkte sinkt also leicht; eine frühere Vermutung, die Ausbeute würde steigen, hat sich nicht bestätigt.

## [1.12.0] - 2026-08-12
### Added
- **Ladeblöcke werden ausgewertet:** BMW liefert je Ladesitzung eine Liste von Blöcken mit Beginn, Ende und Netzleistung. Bisher wurde davon nur die Anzahl übernommen. Das neue `analyse_blocks()` ermittelt daraus die tatsächliche Ladezeit, die Spitzenleistung und die Leistungskurve.
- **AC/DC-Unterscheidung:** Die Spezifikation enthält dafür **kein Feld** — die Ladeart wird aus der höchsten Blockleistung abgeleitet. Wechselstrom endet bauartbedingt bei 22 kW, Gleichstrom beginnt bei rund 50; die Schwelle liegt bei 25 kW und damit in einer leeren Lücke. In der Oberfläche ist die Angabe als Ableitung gekennzeichnet, nicht als Angabe von BMW. Die Zusammenfassung zeigt Vorgänge und Energie nach Ladeart getrennt.
- **Zeitraumfilter (7 / 14 / 30 / 45 Tage):** Die drei kürzeren Zeiträume filtern den bereits geholten Verlauf im Browser und kosten **keinen** zusätzlichen BMW-Abruf. Nur 45 Tage holt einen eigenen, separat zwischengespeicherten Zeitraum. Die Kennzahlen über der Tabelle rechnen dabei mit dem sichtbaren Ausschnitt statt mit dem gesamten Abruf.
- **Detailzeile je Ladevorgang:** Ein Klick auf eine Zeile zeigt Spitzenleistung, tatsächliche Ladezeit, Anzahl der Ladeblöcke, Kilometerstand und ob vorkonditioniert wurde. Diese Angaben kamen bereits von BMW, wurden aber nie dargestellt.

### Fixed
- **Die angezeigte Ladeleistung war systematisch zu niedrig:** Sie wurde als Energie geteilt durch die **Gesamtdauer** berechnet — also einschließlich der Zeit, in der das Fahrzeug voll am Kabel stand. Am Produktivsystem gemessen, beides dieselbe 11-kW-Wallbox:

  ```
  09.08.  12,77 kWh  238 min  ->  3,22 kW  bei 72 Ladeblöcken
  10.08.  16,46 kWh   86 min  -> 11,48 kW  bei  1 Ladeblock
  ```

  Die 3,22 kW sind kein Messwert, sondern ein Artefakt der Rechnung. Die Spalte zeigt jetzt die Leistung **während des Ladens**; der Wert über die gesamte Steckzeit steht als Hinweis am Feld. Die Spalte „Dauer" heißt entsprechend „Am Kabel".

### Note
Ladeverluste lassen sich **nicht** berechnen. Das Feld `energyDecreaseHvbKwh` wäre die batterieseitige Energie, ist aber in allen zehn geprüften Sitzungen leer — BMW liefert es für dieses Fahrzeug nicht.

Die AC/DC-Schwelle ist an keiner Schnellladung geprüft: Alle vorliegenden Sitzungen sind Heimladungen bei rund 11 kW. Die Einordnung folgt der Spezifikation, nicht einer Messung. Die Leistungskurve wird unter `power_curve` mitgeliefert — damit lassen sich die Feldnamen nach dem Ausrollen am Produktivsystem prüfen, ohne einen weiteren Abruf zu verbrauchen.

## [1.11.1] - 2026-08-12
### Fixed
- **Die GPS-Prüfung der Zeitstempel-Diagnose konnte nie anschlagen:** Sie verglich, ob Breite und Länge **derselben Nachricht** dieselbe Messzeit tragen. Die erste Messung am Produktivsystem zeigte: 93 Nachrichten, 93 Datenpunkte — **BMW schickt jede Metrik einzeln**. Breite und Länge treffen deshalb nie zusammen ein, `pairs` wäre dauerhaft 0 geblieben. Verglichen wird jetzt der zuletzt gesehene Zeitstempel je Komponente, also über Nachrichtengrenzen hinweg. Der Wert `partial_messages` entfällt — er hätte bei jeder GPS-Nachricht hochgezählt und nichts ausgesagt.
- Neue Felder unter `bmw_timestamps.gps`: `latitude_messages`, `longitude_messages`, `observations`, `equal_timestamp` sowie `spread_seconds` mit Minimum, Maximum und Median.

### Note
Der Befund erklärt das Grundproblem schärfer als bisher formuliert. Weil beide Hälften getrennt eintreffen, muss die Bridge sie über Nachrichtengrenzen hinweg zusammensetzen — genau dafür existieren die Flags und die Toleranz im `LocationTracker`, und genau daraus entstanden in frühen Versionen die **rechtwinkeligen Linien** auf der Karte: ein neuer Breitengrad, kombiniert mit einem alten Längengrad, ergibt Bewegung auf nur einer Achse und dann einen Knick.

`GPS_MAX_TIMESTAMP_DELTA=30` begrenzt diesen Fehler bislang nur — bei 130 km/h sind 30 Sekunden über ein Kilometer Versatz auf einer Achse, also genau die Größenordnung dieser Knicke. Tragen beide Hälften denselben BMW-Zeitstempel, ersetzt ein exakter Vergleich die Toleranz, und der Fehler ist konstruktiv ausgeschlossen statt begrenzt. Ob das so ist, misst diese Fassung.

Erste Zahlen aus dem Stand (v1.11.0, 93 Nachrichten): Abdeckung 100 %, kein Datenpunkt ohne Zeitstempel, Verzögerung zwischen Messung und Ankunft 1,57 s bis 5,01 s bei einem Median von 3,39 s.

## [1.11.0] - 2026-08-12
### Added
- **Zeitstempel-Diagnose:** BMW liefert zu jedem Datenpunkt einen eigenen, sekundengenauen Zeitstempel (`{"timestamp": "2026-08-11T10:49:00Z", "value": 11260, "unit": "W"}`). Die Bridge verwirft ihn und stempelt jeden Wert mit der **Ankunftszeit**. Das hat Folgen: Der Schutz `GPS_MAX_TIMESTAMP_DELTA` vergleicht, wann Breite und Länge *eingetroffen* sind, nicht wann sie gemessen wurden — gegen eine verspätet gelieferte Altposition hilft er deshalb nicht, denn die kommt als stimmiges Paar an. Genau daraus entstand der Ausreißer vom 25.07., den v1.10.0 nur in der Anzeige abfängt.
- **Neues Modul `lib/timestamp_probe.py`:** Zählt im Normalbetrieb mit, wie verlässlich BMWs Angabe ist — wie viele Datenpunkte einen Zeitstempel tragen, wie weit Messzeit und Ankunftszeit auseinanderliegen (Minimum, Maximum, Median), ob Breite und Länge derselben Nachricht dieselbe Messzeit tragen, und wie oft BMW unvollständige Teilpakete schickt. Die Zahlen stehen unter `/api/status` im Feld `bmw_timestamps`.
- **Warnung bei fehlendem Zeitstempel:** Der einzige Fall, der die geplante Umstellung kippen würde, landet als `WARNING` im Log — je Messwert genau einmal, damit ein Dauerfehler das Logfile nicht füllt.

### Fixed
- **Beworbene Funktion entfernt, die es seit v1.8.5 nicht mehr gibt:** In der Funktionsliste beider READMEs stand ein „Automatischer Container-Fallback", der bei `No active container found` selbsttätig einen Telemetrie-Container anlegt. Im Code existiert davon nichts — der REST-Telemetrie-Abruf samt Container-Verwaltung wurde in v1.8.5 entfernt, die Werbung dafür blieb stehen. Nutzer lasen also von etwas, das die Bridge nicht kann.

### Note
**An der Aufzeichnung ändert sich nichts.** Dieses Release misst nur. Der eigentliche Umbau auf BMWs Messzeit folgt erst, wenn belastbare Zahlen vorliegen — dann lässt sich die 30-Sekunden-Toleranz aus v1.8.3 durch einen exakten Vergleich ersetzen. Tests halten fest, dass `location_tracker.py` unberührt bleibt und das Diagnosemodul weder schreibt noch Netzverkehr erzeugt.

Die Diagnose ist als Werkzeug auf Zeit gedacht. Ist die Frage beantwortet, kann sie wieder verschwinden.

## [1.10.2] - 2026-08-12
### Added
- **Dependabot ist aktiv:** Bislang war im Quell-Repository weder eine Sicherheitswarnung noch ein Versionsupdate eingeschaltet — bei neun fest gepinnten Paketen in `requirements.txt` hieß das, dass eine Version so lange stehen bleibt, bis sie jemandem auffällt. Eingeschaltet wurden die Sicherheitswarnungen samt automatischer Sicherheits-PRs in den Repository-Einstellungen sowie die regelmäßigen Versionsupdates über die neue `.github/dependabot.yml`.
- **Drei überwachte Ökosysteme:** `pip` (requirements.txt), `github-actions` (acht Aktionen in zwei Workflows) und `docker` (Basisimage). Alle wöchentlich, montags früh. Patch- und Minor-Updates kommen gebündelt in je einem Pull Request — neun einzelne pro Woche wären für ein Privatprojekt unbrauchbar. Major-Updates bleiben einzeln, weil sie eine bewusste Entscheidung verlangen.
- **Neue Tests `tests/test_dependabot_config.py`:** Sie prüfen nicht die YAML-Syntax, sondern die Verbindung zwischen Konfiguration und Wirklichkeit. Wer künftig ein neues Manifest hinzufügt und Dependabot dabei vergisst, bekommt einen roten Test statt jahrelang unbemerkt veraltender Abhängigkeiten — dasselbe Muster wie in `test_api_budget.py`. Zusätzlich abgesichert ist, dass die CI bei Pull Requests die **vollständige** Testsuite ausführt und der Veröffentlichungs-Workflow nicht auf Pull Requests reagiert.

### Note
Sprünge des Python-Basisimages (3.11 → 3.12 und höher) sind bewusst ausgenommen. Ein solcher Pull Request würde nur die Zeile im `Dockerfile` ändern, während `ci.yml` `python-version` weiterhin fest auf `'3.11'` hält — die Tests liefen gegen die alte Laufzeit, und die Abweichung fiele erst im Betrieb auf. Ein Wechsel der Python-Version gehört in einen eigenen Branch mit beiden Änderungen.

Im öffentlichen Distributions-Repository bleibt Dependabot aus: Dort liegen nur README, LICENSE, `docker-compose.yml` und `example.env` — kein Manifest mit versionierten Abhängigkeiten, das sich überwachen ließe.

## [1.10.1] - 2026-08-11
### Fixed
- **Geschwindigkeitsschwelle des Standortverlaufs von 200 auf 350 km/h angehoben:** Der Wert war an der falschen Größe gemessen — an dem, was dieses eine Fahrzeug in 30 Tagen gefahren ist, statt an dem, was ein Auto überhaupt kann. Der schnellste Serien-BMW liegt bei rund 305 km/h. Eine schnelle Autobahnfahrt wäre dadurch in Schnipsel zerlegt worden, und zwar ausgerechnet bei dem Fahrer, der sie fährt.
- **Die Korrektur kostet nichts:** Zwischen der schnellsten echten Fahrt (176,9 km/h) und dem langsamsten Messfehler (376,7 km/h) liegt in 30 Tagen Produktivdaten kein einziger Wert. 200, 250, 300 und 350 km/h ergeben exakt dieselben fünf Trennungen; gegen die Echtdaten geprüft bleibt das Ergebnis Zahl für Zahl gleich (3 verworfene Punkte, 308 Abschnitte, 307 Lücken). Erst ab 400 km/h würde ein echter Ausreißer durchrutschen. Die Schwelle liegt jetzt mitten im leeren Band und muss keinen Grenzfall entscheiden.
- **Zahlendreher in der Herleitung:** In v1.10.0 stand als höchste plausible Geschwindigkeit 137,8 km/h. Der Wert stammte aus einer nach *Distanz* sortierten Liste, nicht nach Geschwindigkeit. Richtig sind 176,9 km/h. Korrigiert in `lib/track.py` und in den Tests.

### Note
Die fehlende Millisekunden-Auflösung von BMWs Zeitstempeln wurde als möglicher Hinderungsgrund für den geplanten Umbau der Aufzeichnung geprüft und verworfen: Von 10.384 Punktpaaren liegen 40 unter einer Sekunde, davon haben sich drei überhaupt bewegt — um höchstens 4,4 Meter. In 30 Tagen enthalten fünf Sekunden mehr als einen Punkt. Siehe `ROADMAP.md`.

## [1.10.0] - 2026-08-11
### Added
- **Der Standortverlauf behauptet nichts mehr, was er nicht weiß:** Die Karte zog bisher eine einzige durchgehende Linie durch alle Punkte des Zeitraums — auch über Stunden ohne Daten hinweg. Der Verlauf wird jetzt an Meldelücken in Abschnitte getrennt. Wo die gefahrene Strecke unbekannt ist, steht eine **gestrichelte** Verbindung mit einem Hinweis, wie lange die Daten fehlen und wie weit die Luftlinie ist. Unter der Karte steht eine Zeile wie „61 Punkte in 4 Abschnitten · 1 Lücke mit unbekannter Strecke (gestrichelt)".
- **Neues Modul `lib/track.py`:** Zerlegt den Verlauf und erkennt Fehlpositionen. Die Schwellwerte sind aus 30 Tagen Produktivdaten abgeleitet (10.385 Punkte), nicht geschätzt: 89 % aller Punkte liegen unter zwei Minuten auseinander, die größte Lücke *während* einer Fahrt bei sechs Minuten — getrennt wird daher erst ab zehn Minuten. Die höchste plausible Geschwindigkeit in diesem Zeitraum betrug 137,8 km/h; ab 200 km/h gelten zwei Punkte als nicht zusammengehörig.

### Fixed
- **Fehlpositionen verzerren die Karte nicht mehr:** Am 25.07. sprang die gemeldete Position für zwölf Sekunden 198 km nach Salzburg und auf den Meter genau zurück — 301.390 km/h für den Rückweg. Das zeichnete nicht nur ein V quer durch Österreich, sondern zog auch den Kartenausschnitt auf 224 km Breite auf, sodass von der eigentlichen Fahrt ein Punkt übrig blieb. Solche kurzen, weit entfernten Ausflüge, hinter denen der Verlauf wieder zusammenschließt, werden nicht mehr gezeichnet. **Gefiltert wird ausschließlich in der Darstellung** — die Datenbank behält jeden aufgezeichneten Punkt, und ältere Aufzeichnungen profitieren sofort.
- Eine echte Ortsveränderung wird davon nicht berührt: Wer nach einer Datenlücke woanders auftaucht und dort bleibt, ist gefahren. Verworfen wird nur, was hin- und zurückspringt. Am Anfang und Ende eines Zeitraums wird nie verworfen — ohne Nachbarn auf beiden Seiten lässt sich das nicht entscheiden.

### Changed
- **`GET /api/location-history` liefert ein Objekt statt einer flachen Liste:** `{segments, gaps, dropped, points}`. Wer den Endpunkt direkt anzapft, muss das nachziehen. Die mitgelieferte Oberfläche ist angepasst.

### Note
Die Aufzeichnung selbst bleibt unverändert. Dabei ist aufgefallen, dass BMW zu **jedem** Datenpunkt einen eigenen, sekundengenauen Zeitstempel mitliefert (`{"timestamp": "2026-08-11T10:49:00Z", "value": 11260, "unit": "W"}`), die Bridge für ihre GPS-Logik aber die Ankunftszeit verwendet. Der vorhandene Schutz `GPS_MAX_TIMESTAMP_DELTA` vergleicht deshalb, wann Breiten- und Längengrad *eingetroffen* sind — gegen eine verspätet gelieferte Altposition hilft das nicht. Das ist ein eigenes Thema und folgt in einem späteren Release.

## [1.9.5] - 2026-08-11
### Fixed
- **Erstanmeldung schlug bei jeder Neuinstallation fehl (Issues #1 und #2):** Die Bridge entschied allein anhand der *Existenz* der Token-Datei, ob sie den Anmeldeprozess startet. Die Installationsanleitung wies aber an, die Datei vorab mit `echo "{}" > bmw_tokens.json` anzulegen — sie wird als Volume in den Container gehängt und muss dafür vorhanden sein. Jede Neuinstallation landete dadurch sofort im Service-Betrieb: Der Anmeldedialog erschien nie („the link to the bmw homepage does not show"), und der Start endete mit „Invalid Access Token". Maßgeblich ist jetzt der **Inhalt** der Datei, nicht ihr Vorhandensein — das neue Modul `lib/tokens.py` erkennt fehlende, leere, unlesbare und tokenlose Dateien gleichermaßen als „noch keine Anmeldung".
- **Anleitung legt keine gefüllte Token-Datei mehr an:** Statt `echo "{}"` steht dort nun `touch bmw_tokens.json`. Die Datei muss weiterhin angelegt werden — fehlt sie, erzeugt Docker beim Einhängen ein *Verzeichnis* an ihrer Stelle. Ein Test wacht darüber, dass die alte Anweisung nicht zurückkehrt.

### Note
Ein **abgelaufener** Refresh-Token gilt bewusst weiterhin als brauchbar. Würde er als unbrauchbar gelten, spränge die Bridge im Dauerbetrieb in den interaktiven Anmeldedialog — im Container genau das Falsche. Abgelaufene Token bleiben Sache der Anmeldung.

## [1.9.4] - 2026-08-11
### Changed
- **Releases entstehen jetzt in beiden Repositories:** Der Veröffentlichungs-Workflow legte ein GitHub-Release bislang nur im öffentlichen Repository an. Die Release-Übersicht des Quell-Repositorys hing dadurch seit Juni auf v1.8.4 fest, obwohl längst v1.9.3 lief. Der Workflow erzeugt das Release nun zusätzlich im Quell-Repository — dort genügt der Standard-Token, er braucht lediglich Schreibrechte auf `contents`. Die dreizehn fehlenden Releases ab v1.8.5 wurden nachgetragen; `v1.8.6` blieb bewusst aus, da dieser Tag aus der in v1.8.7 bereinigten Kollision stammt und keinen eigenen CHANGELOG-Eintrag besitzt.

## [1.9.3] - 2026-08-11
### Fixed
- **Ladeverlauf funktioniert:** Der Abruf schlug seit v1.9.0 durchgehend mit `CU-401` fehl. Ursache war weder das Zahlenformat noch die Zeitzone, sondern **der zu große Zeitraum**: BMW lehnt Anfragen über mehr als rund anderthalb Monate ab, ohne das zu dokumentieren oder in der Fehlermeldung zu erwähnen. Am Produktivsystem gemessen: 90 und 60 Tage abgelehnt, 45 und 30 Tage akzeptiert. Der Standardzeitraum liegt jetzt bei 30 Tagen, das Maximum bei 45. Größere Werte weist die Bridge selbst ab, statt einen Abruf zu verbrauchen, der sicher scheitert.
- **Lesbare Energiewerte:** BMW liefert die geladene Energie in voller Maschinenpräzision — im Livebetrieb etwa `22.979991912841797 kWh`. Die Werte werden jetzt auf zwei Nachkommastellen gerundet, auch in den Summen.

## [1.9.2] - 2026-08-10
### Added
- **Fehlerantworten von BMW sind jetzt diagnostizierbar:** Bei einem Fehler protokolliert die Bridge zusätzlich die angefragte Adresse und die unveränderte Antwort von BMW. Bisher wurde nur die übersetzte Meldung geschrieben — bei `CU-401` („Übergabewert ungültig") ließ sich dadurch weder erkennen, was gesendet wurde, noch was BMW konkret bemängelte. Beim Tageslimit `CU-429` bleibt es bei der kurzen Meldung, dort ist die Ursache eindeutig.

### Changed
- **Zeitstempel des Ladeverlaufs mit Millisekunden:** Der Abruf schlug im Produktivbetrieb weiterhin mit `CU-401` fehl, obwohl seit v1.9.1 gültiges ISO 8601 gesendet wurde (`2026-05-12T11:24:07Z`). Die Swagger-Spezifikation nennt lediglich `format: date-time` ohne weitere Angabe. Nachdem Unix-Sekunden (v1.9.0) und sekundengenaue Zeitstempel (v1.9.1) beide abgelehnt wurden, folgt nun die bei Java-Backends übliche Schreibweise mit Millisekunden: `2026-05-12T11:24:07.688Z`. **Das ist eine begründete Annahme, keine gesicherte Erkenntnis** — sollte auch sie fehlschlagen, steht der genaue Grund dank der neuen Protokollierung im Log.

## [1.9.1] - 2026-08-10
### Fixed
- **Ladeverlauf lieferte `CU-401`:** Die Zeitraum-Parameter `from` und `to` wurden als Unix-Sekunden übergeben. Die Swagger-Spezifikation gibt für beide jedoch `string` mit `format: date-time` an — BMW erwartet ISO 8601. Der Abruf schlug dadurch bei jedem Versuch mit „Ein Übergabewert der Anfrage war ungültig" fehl. Die Zeitstempel werden jetzt als `2026-05-12T11:05:00Z` in UTC gesendet.
- **`[object Object]` in der Reifenanzeige:** Für Reifen, die keine Daten übertragen, liefert BMW nur das Label ohne Wert (`{"label": "Production date"}`). Der Ausdruck `feld.value || feld` fiel dadurch auf das Objekt selbst zurück, das in der Oberfläche als `[object Object]` erschien — betroffen waren „DOT (Woche/Jahr)" und „Montiert am". Eine neue Hilfsfunktion `pickFieldValue()` liefert in diesem Fall `null`, sodass wie bei den übrigen Feldern ein Strich erscheint.
- **Irreführende Beschriftung „Profiltiefe":** Darunter stand ein Wert wie „Service in 22.300 km". BMW liefert keine Profiltiefe in Millimetern; das Feld `tyreWear` bezeichnet die Restlaufleistung bis zum nächsten Service. Die Beschriftung lautet jetzt „Service fällig in".

## [1.9.0] - 2026-08-10
### Added
- **Neuer Tab „Ladeverlauf":** Zeigt die Ladevorgänge des Fahrzeugs aus der BMW-Schnittstelle `chargingHistory` — Beginn, Ort, geladene Energie, Ladestand von/bis, Dauer und durchschnittliche Ladeleistung. Darüber stehen die Summen für den Zeitraum (standardmäßig 90 Tage). Die Durchschnittsleistung liefert BMW nicht mit; sie wird aus Energie und Dauer berechnet. Auf schmalen Bildschirmen scrollt die Tabelle innerhalb ihres Bereichs, die Seite selbst bleibt ruhig.
- **Fahrzeugliste im Bereich „Settings & Logs":** Zeigt über `vehicles/mappings` alle dem BMW-Konto zugeordneten Fahrzeuge samt Rolle. Das ist mehr als Zierde: Die CarData-Schnittstelle liefert Daten ausschließlich für Fahrzeuge, bei denen man **Hauptnutzer** (PRIMARY) ist — ein Fehler `CU-104` hat hier häufig seine Ursache, und die Liste macht das auf einen Blick sichtbar.
- **Neues Modul `lib/charging.py`:** Bereitet die tief verschachtelte BMW-Antwort in eine flache Form auf. Fehlende Angaben — je nach Fahrzeug und Ladevorgang liefert BMW nicht jedes Feld — führen zu leeren Werten statt zu einem Fehler. Fehlt die aufbereitete Ortsangabe, wird auf den öffentlichen Ladepunkt zurückgegriffen.

### Changed
- **Budgetprüfung deckt neue Endpunkte automatisch ab:** Beide Abrufe sind 24 Stunden zwischengespeichert und kosten damit je einen Abruf pro Tag; der Gesamtverbrauch liegt bei 28 der 50 erlaubten Aufrufe. Ein neuer Test schlägt fehl, sobald ein zwischengespeicherter Endpunkt hinzukommt, ohne in der Budgetrechnung berücksichtigt zu werden.

### Fixed
- **Pfadschreibweise geklärt:** Der Integration Guide schreibt die fahrzeugbezogenen Endpunkte als `/customer/vehicles/…` (Einzahl), die maßgebliche Swagger-Spezifikation dagegen durchgängig als `/customers/vehicles/…` (Mehrzahl). Die Bridge verwendet die Mehrzahl und liegt damit richtig; die offene Frage aus v1.8.12 ist beantwortet.

## [1.8.12] - 2026-08-10
### Fixed
- **Fehler der BMW-API werden im Klartext erklärt:** Die Bridge reichte Fehlermeldungen bisher als Rohtext weiter (`BMW API returned error: {"exveErrorId":"CU-429"}`). Das ist besonders tückisch, weil BMW das erschöpfte Tagesbudget als **HTTP 403** ausliefert — also mit demselben Statuszeichen wie eine fehlende Berechtigung. Genau diese Verwechslung führte im Juni dazu, den REST-Telemetrie-Abruf für unbrauchbar zu halten und in v1.8.5 zu entfernen, obwohl lediglich das Tageslimit erreicht war. Das neue Modul `lib/bmw_api_errors.py` übersetzt alle 19 dokumentierten Fehlercodes (`CU-100` bis `CU-503`, Integration Guide Kapitel 3.4) in verständliche Meldungen samt Hinweis auf die Ursache.
- **Irreführende Beschriftung im SmartMaintenance-Tab:** Der Aktualisieren-Knopf meldete bei jedem Fehler pauschal „Gesperrt (Auth-Fehler)" — auch beim reinen Tageslimit. Er unterscheidet jetzt zwischen „Tageslimit erreicht" und „Nicht verfügbar", und der genaue Grund erscheint als Hinweis im Tab statt nur in der Browser-Konsole.

### Added
- **Buchführung über das Tagesbudget:** Ein Zähler protokolliert die verbrauchten API-Aufrufe und setzt sich um 00:00 UTC zurück, passend zur Vorgabe von BMW. Ist das Budget aufgebraucht, antwortet die Bridge sofort mit einer Erklärung, statt einen Aufruf abzusetzen, der ohnehin abgelehnt würde. Meldet BMW ein `CU-429`, obwohl der eigene Zähler niedriger steht — etwa nach einem Neustart der Bridge oder wenn ein zweiter Client dasselbe Konto nutzt — korrigiert sich der Stand selbst. Der aktuelle Verbrauch steht unter `/api/status` im Feld `api_rate_limit` bereit.
- **43 weitere automatisierte Tests** für Fehlerübersetzung, Budgetzähler und die Verdrahtung beider. Die HTTP-Aufrufe sind durchgehend simuliert; für die Tests geht keine einzige Anfrage an BMW.

## [1.8.11] - 2026-08-07
### Fixed
- **Versionsanzeige stimmt wieder:** Die Versionsnummer wurde an drei Stellen gepflegt und war auseinandergelaufen — `main.py` meldete `1.8.1`, der Badge im Dashboard `v1.8.7`, während tatsächlich v1.8.10 lief. Der Badge wird jetzt zur Laufzeit aus `__version__` über `/api/status` bezogen, statt fest im HTML zu stehen. Damit gibt es nur noch eine Quelle, und ein Test stellt sicher, dass `main.py`, CHANGELOG und README dauerhaft übereinstimmen.
- **Falscher Image-Name in der Entwickler-Dokumentation:** Die `README.md` des Quell-Repositorys verwies an zwei Stellen auf `ghcr.io/bausi2k/bmw-python-streaming-mqtt-bridge:latest`. Unter diesem Namen existiert kein Image — es heißt `bmw-mqtt-bridge`. Ein `docker compose pull` nach dieser Anleitung wäre fehlgeschlagen.
- **`EXPOSE` im Dockerfile korrigiert:** Die Angabe stand auf Port 8000, die Anwendung lauscht aber auf 8999.

### Added
- **Gesundheitsprüfung für den Container:** Ein `HEALTHCHECK` fragt minütlich die eigene Status-Schnittstelle ab. Docker erkennt damit eine hängende Bridge, statt den Container weiterhin als gesund zu führen. Umgesetzt mit Bordmitteln von Python, da das schlanke Basis-Image weder `curl` noch `wget` enthält.

### Removed
- **Toter Code entfernt:** Die Konstanten `AUTH_BASE_URL`, `MQTT_URL` und `MQTT_PORT` sowie der Import von `requests` in `main.py` waren Überbleibsel des in v1.8.5 entfernten REST-Abrufs und wurden nirgends mehr verwendet. Ebenso entfällt die Methode `run_token_monitor()` in `lib/bmw_cardata.py`: Sie wurde nie aufgerufen und duplizierte die Logik des Token-Refresh-Threads. Die verwaiste Datei `example` — eine Kopie eines GitHub-Workflows ohne Dateiendung — ist ebenfalls entfernt.
- **Kein stilles Beenden mit Erfolgsmeldung mehr:** Drei Stellen nutzten `exit()` statt `sys.exit(1)` und meldeten damit Exit-Code 0. Docker und systemd werteten einen Fehlstart dadurch als sauberes Beenden; bei `Restart=on-failure` unterblieb der Neustart.

### Changed
- **Hinweis zum `USER` im Container:** Bewusst weiterhin kein fester Nicht-root-Benutzer. Die Bridge schreibt in eingehängte Volumes (`bmw_tokens.json`, `logs/`, `data/`), die auf dem Host einem bestimmten Benutzer gehören; eine feste UID im Image würde dort zu Schreibfehlern führen. Für den vorgesehenen Betrieb im privaten Heimnetz überwiegt das Risiko den Gewinn.

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

