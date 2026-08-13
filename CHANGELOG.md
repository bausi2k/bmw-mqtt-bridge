# Changelog

[![Buy Me A Coffee](https://cdn.buymeacoffee.com/buttons/v2/default-yellow.png)](https://www.buymeacoffee.com/bausi2k)

Alle nennenswerten Änderungen an diesem Projekt werden in dieser Datei dokumentiert.

Das Format orientiert sich an [Keep a Changelog](https://keepachangelog.com/de/1.0.0/),
und dieses Projekt folgt [Semantic Versioning](https://semver.org/lang/de/).

> Die Historie beginnt mit v1.8.0. Ältere Einträge betreffen überwiegend interne
> Umbauten ohne Auswirkung auf den Betrieb der Bridge.

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

