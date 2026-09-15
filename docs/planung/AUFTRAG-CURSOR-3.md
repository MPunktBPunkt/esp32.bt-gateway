# Anweisung 3 — Hub-Contract, Flash-Budget und Pflichtenheft V2.0

**Adressat:** zweite Cursor-Instanz, die im Repo `esp32.bt-gateway` arbeitet
**Vorgänger:** [AUFTRAG-CURSOR-2.md](AUFTRAG-CURSOR-2.md) — **vollständig umgesetzt**, siehe Kapitel 0
**Stand der Analyse:** Planung **V1.2** (Commits `00aedcd`…`6f80aba`) · `iobroker.esp-hub` **v0.5.8, Adaptercode gelesen** · `ESP32.esp-hub` 1.7.0 · `esp32.heartrate` (HubClient-Referenz) · `pidrive` v0.11.127
**Erstellt:** 2026-09-15

---

## 0. Einordnung: V1.2 ist angekommen

Anweisung 2 wurde in sechs Commits umgesetzt. Der Stand ist gut und wird hier **nicht**
in Frage gestellt:

- `AVRCP-MOEGLICHKEITEN.md` mit Rollenbild, drei Kanälen und den Ausbaustufen S1–S3
- `PHASE-0-MESSPLAN.md` mit Phase −1 auf dem Pi **vor** jeder ESP-Firmware
- `PFLICHTENHEFT.md` V1.2: §2.2 (F4 / Weg F), §2.3 (Phase −1), §2.9 (Browsing),
  §2.10 (Stackabhängigkeit), §2.16, §2.20, **§2.24 Menü-Transport**
- `OFFENE-PUNKTE.md`: A1 entschärft, A17–A20 neu, R18–R23 neu
- `REVIEW-V1.2.md`, `BETRIEBSMODI.md` (Weg E/F), `KONZEPT.md`, `STATE.md`, `README.md`

Besonders richtig: die Umkehr der Reihenfolge (erst messen, dann Stack, dann Bauen) und
die Erkenntnis, dass S3 der eigentliche strategische Grund für das Gateway ist.

**Zwei Dinge fehlen — beide betreffen nicht die Bluetooth-Seite, sondern die Frage, wie
die Firmware überhaupt auf das Gerät kommt und wie groß sie werden darf:**

1. Der Hub-Contract war bisher aus `Schnittstellen.md` abgeleitet. Diese Doku steht auf
   **v0.1.0**, der Adapter auf **v0.5.8** — sie ist an den entscheidenden Stellen falsch
   oder unvollständig. Kapitel 1 ersetzt sie durch am Code verifizierte Befunde.
2. Das **Flash-Budget** fehlt in der Planung vollständig. Es ist aber eine harte
   Nebenbedingung von A17: ein Stack, der AVRCP kann, aber kein Dual-OTA mehr zulässt,
   kostet genau die Kabellosigkeit, die der Owner will. Kapitel 2.2 zieht das nach.

Was weiterhin offen bleibt: **Q1–Q5** aus Anweisung 2 sind unbeantwortet. Kapitel 9
ergänzt Q6–Q11. Keine davon selbst beantworten.

---

## 1. Verifizierte Hub-Fakten

Quelle jeweils `iobroker.esp-hub/main.js` (v0.5.8). **Im Zweifel gilt der Code, nicht
`Schnittstellen.md`.**

| # | Befund | Quelle | Konsequenz |
|---|--------|--------|------------|
| **H-F1** | OTA ist **Pull, nicht Push**. Der Adapter legt `otaUrl` in die Antwort auf `POST /api/register`; der ESP lädt das Image selbst per `GET` über **HTTP ohne TLS**. | `main.js:292–302`, `1147–1156` | In IDF `esp_https_ota` mit `CONFIG_OTA_ALLOW_HTTP=y` oder `esp_http_client` + `esp_ota_*`. Kein ArduinoOTA, kein espota.py, kein Port 3232. |
| **H-F2** | **Die `otaUrl` ist einmalig.** Direkt nach dem Einfügen in die Antwort löscht der Adapter den State. Ein gesendeter Heartbeat gilt als Auslieferung — unabhängig vom Erfolg. | `main.js:298–302` | **Kollidiert frontal mit „OTA nicht während `STREAMING`".** Siehe 2.3. Die Familien-Referenz `heartrate` hält die URL nur in RAM (`HubClient.cpp:44–47,77–81`) und verliert sie bei Reset — dieses Verhalten **nicht** kopieren. |
| **H-F3** | `GET /api/ota/check?mac=…` liest denselben State, **löscht ihn aber nicht**. | `main.js:1135–1144` | Idempotenter Kanal, brauchbar als Nachfragepunkt — solange der Heartbeat die URL noch nicht konsumiert hat. |
| **H-F4** | Der Gerätename wird **nur beim ersten Heartbeat** übernommen (`d.name = d.name \|\| body.name`). Späteres `name` im Payload wird ignoriert. | `main.js:249` | Relevant für **A15/A20**: der Name muss bei der Erstregistrierung stimmen. Späteres Umbenennen nur über die Hub-UI. |
| **H-F5** | Die Chip-Familien-Sperre vergleicht die Familie aus dem **Dateinamen** mit der aus `chipModel`. Ist **eine** Seite unbekannt, wird erlaubt („Unknown sides are allowed"). | `main.js:78–95`, `1201–1221` | `chipModel` ist damit die **Schutzfunktion**, nicht Kosmetik. Ohne das Feld ließe sich ein `.esp32s3.bin` auf den Classic-ESP32 pushen — und der S3 hat kein Classic BT (§2.2). |
| **H-F6** | USB-Flash läuft als `esptool --port … --baud … write_flash <addr> <file>` mit **frei wählbarer Adresse, Default `0x0`**. | `main.js:807–830`, `1349` | **Der Erstflash eines IDF-Merged-Images aus der Hub-UI funktioniert ohne Adapteränderung.** Datei wählen, `0x0` stehen lassen. |
| **H-F7** | Die UI rechnet den Flash-Balken als `freeSketch / 1966080`. Der Nenner ist **hart verdrahtet** — 1 966 080 B = `0x1E0000` = die Arduino-`min_spiffs`-App-Partition. | `main.js:2349` | Ein OTA-Slot von genau `0x1E0000` macht die Anzeige korrekt. Siehe 2.1. |
| **H-F8** | `ios` ist beliebiges JSON und landet als **ein** String-State (`devices.<MAC>.ios`), wird in der Gerätekarte aber als Liste gerendert. | `main.js:260–261, 280` | **Der Gateway erscheint mit allen KPIs im Hub, ohne eine Zeile Adaptercode.** Einzelne States pro KPI nur mit Adaptererweiterung → Q9. |
| **H-F9** | `/api/register` hat **keine Authentifizierung**; MAC wird auf 12 Hex-Zeichen normalisiert. | `main.js:21–23, 236–238` | Reines Heimnetz-Protokoll. Über diesen Kanal **keine** sicherheitsrelevanten Kommandos annehmen. Der PSK aus §2.15 gilt für PDAP, nicht für den Hub. |

**Damit ist A9 beantwortbar.** Der Punkt steht auf „Anforderung gesetzt, Umsetzungsweg
IDF-Client" — der Weg ist jetzt nicht mehr nur plausibel, sondern gegen den Adaptercode
geprüft: Erstflash per USB aus der Hub-UI (H-F6), danach Feld-Updates per WLAN (H-F1),
**ohne Änderung am ioBroker-Adapter**. A9 kann auf „bestätigt" gehen, mit Verweis auf
dieses Kapitel.

Die Einschränkung liegt nicht im Contract, sondern in der Reichweite — **Kapitel 3**.

---

## 2. Verbindliche Konsequenzen für die Firmware

### 2.1 Partitionstabelle festlegen — als Spezifikation, noch nicht als Datei

Wichtig: V1.2 verbietet zu Recht `main/`/`components/` vor A17 (`PHASE-0-MESSPLAN.md`
§1, Pflichtenheft §2.16). Das gilt auch hier. Die Tabelle wird jetzt **entschieden und
im Pflichtenheft festgeschrieben**; die Datei `partitions.csv` entsteht erst mit dem
Phase-0-Skeleton. Grund für das Vorziehen der Entscheidung: Offsets später zu ändern
kostet im Feld einen USB-Flash — also genau die Kabellosigkeit, um die es geht.

Aufnehmen in Pflichtenheft §2.12a und `OFFENE-PUNKTE.md` Abschnitt D:

```
# Name,     Type, SubType,  Offset,   Size
nvs,        data, nvs,      0x9000,   0x8000
otadata,    data, ota,      0x11000,  0x2000
phy_init,   data, phy,      0x13000,  0x1000
coredump,   data, coredump, 0x14000,  0xC000
ota_0,      app,  ota_0,    0x20000,  0x1E0000
ota_1,      app,  ota_1,    0x200000, 0x1E0000
logs,       data, littlefs, 0x3E0000, 0x20000
```

Rechnung, bitte nachvollziehen und nicht „optimieren":

- `0x3E0000 + 0x20000 = 0x400000` → die Tabelle füllt 4 MB **exakt** aus.
- App-Partitionen müssen 64-KB-ausgerichtet sein: `0x20000` und `0x200000` ✓.
- `0x1E0000` = 1 966 080 B pro Slot → Hub-Flash-Balken korrekt skaliert (H-F7).
- **Kein `factory`-Slot.** Bei Dual-OTA bootet `otadata` per Default `ota_0`.
  Recovery läuft über USB-Merged-Flash (H-F6), nicht über eine Factory-App.
- 32 KB `nvs` wegen WLAN + Hub-Host + PSK + **BT-Bonding-Keys**; bei BTstack zusätzlich
  Link-Key-Ablage. Die Default-24-KB werden hier knapp.
- `logs` (128 KB) deckt die optionale Log-Persistenz aus §2.23. Wird sie nicht
  gebraucht: Partition stehen lassen, **nicht** in die App-Slots umwidmen — sie ist die
  Reserve für 2.2.

### 2.2 Flash-Budget ist ein drittes Kriterium für A17

Aus 2.1 folgt eine harte Grenze: **die App muss unter 1 966 080 Bytes bleiben** —
inklusive Host-Stack, WiFi, TCP/IP, HTTP-Server, eingebetteter WebUI, SBC-Encoder, PDAP
und (bei S3) Menü-Target. Bei Dual-OTA auf 4 MB ist der Rest des Flashs verplant.

A17 wird in `OFFENE-PUNKTE.md` bisher über **Metadaten, Browsing, Aufwand, Risiko**
entschieden. **Ergänze die Matrix um eine Spalte „Codegröße"** und um diesen Satz:
Eine Stack-Entscheidung ohne Größenmessung ist unzulässig, weil BTstack zwar F1/F2
löst, aber zusammen mit WiFi und WebUI das OTA-Budget reißen kann. Die Folge wäre
entweder ein 8/16-MB-Modul — dann ist die Hub-Referenzhardware D1 Mini raus (**A16**,
**Q6**) — oder der Verzicht auf Dual-OTA.

**Wie wird gemessen, ohne Firmware anzulegen?** Nicht mit unserer App — die gibt es
noch nicht und darf es vor A17 nicht geben. Sondern mit **Upstream-Beispielen in einem
Scratch-Verzeichnis außerhalb des Repos**:

| Kandidat | Messobjekt |
|----------|-----------|
| Bluedroid | IDF-Beispiel `bluetooth/bluedroid/classic_bt/a2dp_source`, einmal unverändert, einmal mit aktiviertem WiFi + `esp_http_server` |
| BTstack | ESP32-Port, `a2dp_source_demo` bzw. AVRCP-Target-Demo, gleiche WiFi-Ergänzung |

Ausgewertet wird `idf.py size` / `size-components`: Gesamtgröße, Anteil BT-Stack, Anteil
WiFi/LWIP, **verbleibende Reserve in Prozent von `0x1E0000`**. Ergebnis nach
`docs/planung/FLASH-BUDGET.md` — ein reines Planungsdokument, keine Firmware, damit kein
Konflikt mit Anweisung 2 §5. Nur die Zahlen und die Messmethode dokumentieren, keinen
Code aus den Beispielen ins Repo kopieren.

Unter 20 % Reserve → an den Owner eskalieren, nicht stillschweigend weiterbauen.

**Nebenbedingung, die daraus folgt:** Die WebUI muss **in die App eingebettet** werden
(`EMBED_FILES`/`EMBED_TXTFILES`) und klein bleiben. Kein Asset-Dateisystem, keine
eingebetteten Fonts, keine Chart-Bibliothek als Datei. Falls Diagramme gewünscht sind:
Inline-SVG wie `esp-hub-base`. In §2.12 aufnehmen.

### 2.3 OTA-Zustandsmaschine — der Defer muss die URL überleben

Das ist die wichtigste Implementierungsvorgabe dieses Dokuments, weil hier die
bestehende Anforderung „OTA nur außerhalb `STREAMING`" (§2.12a) auf H-F2 trifft: Der Hub
liefert die URL **einmal** und vergisst sie. Wer den Heartbeat beantwortet und dann
defert, ohne zu sichern, hat das Update **verloren** — im Hub steht „OTA-URL gesendet",
am Gerät passiert nie etwas. Genau das würde der Owner als „OTA funktioniert nicht"
erleben, und es wäre im Log des Hubs nicht als Fehler sichtbar.

**Vorgabe:**

1. Kommt `otaUrl` in der Heartbeat-Antwort → **sofort in NVS** (`pending_ota_url`,
   `pending_ota_since`, `pending_ota_attempts`), **bevor** irgendetwas anderes passiert.
2. Erst danach `ota_allowed()` prüfen. Erlaubt = Zustand **nicht** `STREAMING`
   (Analyzer-Lauf: siehe Q7).
3. Nicht erlaubt → Zustand `OTA_PENDING`, sichtbar in `ios` und lokaler WebUI mit Grund.
   Beim Verlassen von `STREAMING` automatisch nachziehen; nach Reset aus NVS aufnehmen.
4. Erfolgreich → `esp_ota_set_boot_partition` + Reboot. NVS-Eintrag erst nach dem
   Validierungslauf löschen (Punkt 6).
5. Fehlgeschlagen → `attempts++`, Backoff wie bei BT (1 s → 2 s → 5 s → 10 s → 30 s →
   60 s), nach 5 Versuchen aufgeben und Fehler in `ios` melden. **Kein** Dauer-Retry:
   ein wiederholter 1,4-MB-Download während der Fahrt geht direkt auf die Coexistence.
6. **Rollback aktivieren** (`CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE=y`). Die neue App
   ruft `esp_ota_mark_app_valid_cancel_rollback()` **erst**, wenn WLAN steht **und** ein
   Heartbeat erfolgreich beantwortet wurde. Damit ist ein kaputtes Image selbstheilend
   statt USB-pflichtig — das ist die eigentliche Absicherung der Kabellosigkeit.
7. **Merged-Image-Falle hart abfangen.** Das Risiko „Merged- statt App-Bin gepusht"
   (`HUB-INTEGRATION.md` §6) ist prüfbar: Ein IDF-App-Image trägt an **Offset `0x20`**
   das `esp_app_desc_t`-Magicword **`0xABCD5432`** (24 B Image-Header + 8 B
   Segment-Header). Ein Merged-Image hat dort Bootloader-Inhalt. Die ersten 32+ Bytes
   prüfen und **abbrechen, bevor** in die OTA-Partition geschrieben wird. Die Prüfung
   auf `0xE9` allein genügt **nicht** — der Bootloader beginnt ebenfalls mit `0xE9`.
8. Zusätzlich `Content-Length` gegen `esp_ota_get_next_update_partition()->size` prüfen
   und mit klarer Meldung abbrechen, statt im Schreiben zu scheitern.

### 2.4 Heartbeat-Payload — Feldliste mit Begründung

Alle `interval` Sekunden (Default 30) an `POST /api/register`:

| Feld | Wert | Warum verbindlich |
|------|------|-------------------|
| `mac` | STA-MAC, 12 Hex ohne Trenner | Pflicht, sonst `{ok:false}` (`main.js:237`) |
| `name` | z. B. `PiDrive-GW` (A20) | **Nur der erste Wert zählt** (H-F4) |
| `hwType` | `"esp32"` | Chip-Badge + Pinout-Panel |
| `chipModel` | aus `esp_chip_info()`, z. B. `ESP32-D0WD` | **Schutzfunktion gegen S3-Bins** (H-F5). Ohne das Feld ist die Sperre inaktiv. Nicht optional. |
| `version` | SemVer | Versionsvergleich, Release-Nachvollzug |
| `ip`, `rssi`, `uptime`, `freeHeap` | laufend | Familien-Standardfelder |
| `freeSketch` | Reserve im OTA-Slot | Flash-Balken (H-F7), Semantik → Q8 |
| `fwType` | `"bt-gateway"` | Familien-Kennung. Achtung: Adapter v0.5.8 **liest das Feld nicht** — Vorsorge, kein Wirkbetrieb. Nicht darauf verlassen. |
| `ios` | Gateway-KPIs, s. u. | macht den Gateway in der Hub-UI lesbar (H-F8) |

Empfohlener `ios`-Inhalt, bewusst klein (der Payload geht über dieselbe Funkstrecke wie
A2DP):

```json
"ios": {
  "bmw":       { "type": "sensor", "value": "connected" },
  "state":     { "type": "sensor", "value": "STREAMING" },
  "buffer":    { "type": "sensor", "value": 62, "unit": "%" },
  "underruns": { "type": "sensor", "value": 0 },
  "btRssi":    { "type": "sensor", "value": -64, "unit": "dBm" },
  "pdap":      { "type": "sensor", "value": "online" },
  "otaState":  { "type": "sensor", "value": "pending (streaming)" }
}
```

`otaState` gehört ausdrücklich dazu: es ist die einzige Stelle, an der der Owner im Hub
sieht, dass ein Update **wartet** statt verschwunden zu sein (2.3, Punkt 3). Die KPIs
selbst sind eine Teilmenge von §2.13 — nicht neu erfinden, sondern spiegeln.

### 2.5 Release-Artefakte, Namensschema, Checkliste

| Artefakt | Dateiname | Zweck | Adresse |
|----------|-----------|-------|---------|
| Merged | `bt-gateway.<semver>.usb.esp32.bin` | Erstflash + Recovery über Hub-UI | `0x0` |
| App-only | `bt-gateway.<semver>.esp32.bin` | OTA-Push | OTA-Slot |

- Merged mit `idf.py merge-bin` (Raw) erzeugen, nicht händisch zusammenkopieren.
- Beide Namen enden auf `.esp32.bin` → Familien-Parser liefert `esp32` (H-F5) ✓.
  `usb` als Zwischensegment ist für den Parser unschädlich, für den Menschen aber der
  entscheidende Unterschied. **Ersetzt 2.3/7 nicht** — der Dateiname ist Konvention,
  die Firmware-Prüfung ist die Absicherung.
- **Nie** ein `.esp32s3.bin` erzeugen (§2.2: kein S3, kein Classic BT).
- Release-Checkliste nach `docs/RELEASE.md`: beide Bins gebaut, Größe gegen `0x1E0000`
  geprüft, `version` im Heartbeat == Dateiname, Merged nur auf `0x0`, App-Bin einmal per
  OTA gegen ein Testgerät verifiziert.

**Hinweis für §2.22 (Ersteinrichtung):** Ein Merged-Write auf `0x0` überschreibt den
NVS-Bereich mit `0xFF`. WLAN, Hub-Host, PSK und **BT-Bonding zum BMW** sind danach weg
und müssen neu eingerichtet werden — inklusive erneutem Pairing am Fahrzeug. Bei
Recovery gewollt, beim Routine-Update ein guter Grund, OTA zu benutzen. In §2.22 und in
die WebUI-Warnung aufnehmen.

### 2.6 Erstkonfiguration und Fehlersuche ohne Kabel

`esp-hub-base` nutzt WiFiManager mit dem Portal `ESP-Hub-Setup`; für IDF gibt es das
nicht. Baue ein minimales SoftAP-Setup (SSID-Muster der Familie folgen, z. B.
`BT-Gateway-Setup`, WPA2 nach §2.23). Abzufragen: WLAN-SSID/Passwort, **Hub-Host + Port
(Default 8093)**, PDAP-PSK, BT-Gerätename (A15/A20).

Zusätzlich, weil der Hub-Serial-Monitor nur über USB funktioniert: **Logs müssen über
WLAN lesbar sein** — RAM-Ringpuffer in der lokalen WebUI plus Export (§2.23, §2.13).
Ohne das ist „kabellos programmieren" nur die halbe Strecke, weil jede Fehlersuche
wieder am USB-Kabel endet. Das betrifft besonders Phase 0: der Analyzer-Export ist
sowieso gefordert, er muss nur auch den **Systemlog** umfassen.

---

## 3. Neuer Befund: im Auto ist der Hub nicht erreichbar

Der ioBroker läuft zu Hause. Der Gateway hängt im Auto entweder im SoftAP-Modus oder am
Pi (`BETRIEBSMODI.md`) — in beiden Fällen **ohne Route zum Hub**. Der Adapter bettet
zudem seine eigene IP in die OTA-URL ein (`main.js:1216`), die URL ist außerhalb des
Heimnetzes also auch dann wertlos, wenn irgendeine Verbindung bestünde.

Praktisch: **Hub-OTA funktioniert nur im Carport bzw. Heim-WLAN.** Das ist keine
Fehlfunktion, sondern eine Eigenschaft der Topologie — sie muss aber ins Pflichtenheft,
sonst ist die Anforderung „kabellos updaten" zur Hälfte unerfüllt und fällt erst im Feld
auf, typischerweise dann, wenn ein Analyzer-Fix während einer Messfahrt gebraucht wird.

**Konsequenz: zweistufiger Update-Pfad.** Damit wird das lokale Web-OTA vom optionalen
Recovery-Weg (`HUB-INTEGRATION.md` §1/§5, dort H6 „optional") zur **primären
Feld-Update-Strecke**.

| Stufe | Lage | Weg | Status |
|-------|------|-----|--------|
| 1 | Carport / Heim-WLAN | Hub-UI → `otaUrl` → ESP zieht Image | Pflicht (2.3) |
| 2 | Auto, kein Heim-WLAN | PiDrive hält das Image vor und schiebt es per HTTP in das lokale OTA des ESP | **neu, Pflicht** |

Stufe 2 ist billig, weil Pi und ESP für PDAP ohnehin IP-Konnektivität haben. Sie braucht
gateway-seitig genau einen Endpunkt — `POST /ota-upload`, PSK-geschützt, **derselbe
Schreibpfad und dieselben Prüfungen wie 2.3/6–8** — und pi-seitig ein kleines
Arbeitspaket:

- Firmware-Depot auf dem Pi (Image + Version + Prüfsumme), gefüllt aus dem Hub, solange
  Heim-WLAN da ist.
- `pidrivectl gateway firmware fetch|list|push <version>` — passt genau in die
  CLI-Linie, die im PiDrive-Auftrag als bester Debug-Weg unterwegs festgelegt ist.
- Ablehnen, wenn der Gateway `STREAMING` meldet: dieselbe Defer-Regel wie 2.3.

**Dieses Arbeitspaket gehört ins `pidrive`-Repo, nicht hierher.** Hier entsteht **nur**
der Contract (Endpunkt, Auth, Prüfungen, Fehlercodes) in `PIDRIVE-INTEGRATION.md`, plus
ein Vermerk in `OFFENE-PUNKTE.md`, dass die Pi-Seite nachzuziehen ist. Nicht im
Nachbarrepo arbeiten (Kapitel 8).

**Neue Einträge in `OFFENE-PUNKTE.md`:**

- **A21 — Feld-Update-Pfad Stufe 2:** PiDrive als Firmware-Depot. Alternativen:
  (a) Stufe 2 wie beschrieben, (b) nur Carport-Updates und im Auto USB, (c) Gateway
  bekommt im Auto per APSTA doch eine Route nach Hause. Empfehlung (a).
- **R24 — Hub im Fahrzeug nicht erreichbar:** Mitigation = Stufe 2 + lokales Web-OTA
  als Pflicht.
- **R25 — App-Image überschreitet den OTA-Slot:** Mitigation = frühe Messung (2.2),
  eingebettete WebUI, Eskalation ab < 20 % Reserve; Rückfallebene A16/Q6.

---

## 4. Pflichtenheft: V2.0 anlegen, zweischichtig

**Kurzantwort auf die Frage des Owners:** Ja. Das Dokument ist mit V1.2 inhaltlich weit;
was fehlt, ist nicht Text, sondern **Verbindlichkeit**. Der Kopf sagt „Entwurf", und
einige Kapitel hängen an Messungen, die es noch nicht gibt. Komplett freigeben wäre
falsch, alles als Entwurf liegen lassen blockiert aber die längst entschiedenen Teile —
darunter der gesamte Hub-/OTA-Teil, der jetzt am Adaptercode verifiziert ist.

**Entscheidung des Owners: eine Datei mit Marken pro Kapitel**, kein Dateisplit. Jedes
Kapitel trägt im ersten Absatz `[FIX]` oder `[ENTWURF — Gate: …]`.

### 4.1 Teil A — jetzt einfrierbar `[FIX]`

| Kapitel | Warum jetzt entscheidbar |
|---------|--------------------------|
| §2.1 Architektur & Verantwortungsgrenzen | durch die PiDrive-Analyse bestätigt, in V1.2 nachgezogen |
| §2.2 Nicht-Ziele | F4 belegt (kein Sink+Source); S3-Ausschluss technisch zwingend |
| §2.4 Coexistence-Gate (Kriterien, nicht Ergebnis) | Kriterien stehen; nur die Durchführung ist stackabhängig |
| §2.8 PDAP Kanäle/Rollen **ohne** Byte-Layout | Transportentscheidung steht |
| §2.11 Statusmodell, §2.13 KPIs, §2.14 Recovery, §2.15 Security | unabhängig vom BT-Stack |
| **§2.12a Hub & OTA** | **vollständig verifiziert — Ersatztext in Kapitel 5** |
| §2.21 Hardware (Classic ESP32, kein S3) | S3 hat kein Classic BT — nicht verhandelbar |
| §2.22 Ersteinrichtung | mit 2.5/2.6 konkretisiert, inkl. NVS-Verlust beim Merged-Flash |
| **neu:** Partitionstabelle & Flash-Budget | 2.1/2.2 |
| **neu:** zweistufiger Update-Pfad | Kapitel 3 |

### 4.2 Teil B — bleibt Entwurf, mit benanntem Gate

| Kapitel | Gate |
|---------|------|
| §2.3 Phase 0 (ESP-Teil) | Phase −1 abgeschlossen |
| §2.9 AVRCP-Mapping | Phase-0-Messung am realen NBT |
| §2.10 Metadata-Engine | Phase −1 (kommen die Zeilen an?) + A17 |
| **§2.24 Menü-Transport** | **Phase −1 Browsing-Probe** → S1 oder S3; dann A18 |
| §2.16 Task-/Komponentenstruktur | A17 (steht so schon in V1.2) |
| §2.6/2.7 Bitpool und Buffer-Zielwerte | Coexistence-Gate + Hörtest |
| §2.8 Byte-Layout, CRC, Feldbreiten | nach der Audio-Strecke im Labor |
| §2.17 DAB im Gateway-Pfad | A7 + PipeWire-Bridge auf dem Pi |

**Achtung beim Einordnen:** §2.12a wandert nach Teil A, obwohl es OTA in Zustände
einhängt, die erst in Teil B final sind. Das ist beabsichtigt — die Schnittstelle zum
Hub ist fix, nur der Zeitpunkt von `ota_allowed()` hängt an der State Machine. Im Text
so kennzeichnen, damit es nicht als Widerspruch gelesen wird.

### 4.3 Formalia für V2.0

- Titelzeile und Statuszeile widersprechen sich seit V1.0: die H1 sagt
  `Pflichtenheft V1.0`, der Status `Entwurf V1.2`. **Beim Umstellen auflösen**, nicht
  fortschreiben. Neu: `# Pflichtenheft V2.0 — PiDrive Bluetooth Gateway (ESP32)`,
  Status `Teil A verbindlich, Teil B Entwurf`.
- **Änderungshistorie** am Dokumentende (V1.0 → V1.1 → V1.2 → V2.0, je 1–2 Zeilen).
  Sie fehlt bisher und ist der Grund, warum der Stand schwer zu lesen ist.
- Nachzüge aus `REVIEW-V1.1.md`/`REVIEW-V1.2.md`, die inzwischen im Text stehen, dort
  als erledigt markieren statt doppelt zu führen.
- Jedes `[ENTWURF]`-Kapitel nennt **ein** Gate und **einen** Ort für das Ergebnis. Kein
  „wird später entschieden" ohne Zielort.
- `OFFENE-PUNKTE.md` Abschnitt **E (Freigabe-Checkliste)** ist der richtige Ort für die
  Freigabe — **nicht** eine zweite Checkliste im Pflichtenheft anlegen. Ergänze E um:
  „Teil A/B-Marken gesetzt", „Hub-Contract gegen `main.js` verifiziert (A9)",
  „Flash-Budget gemessen (R25)", „Update-Pfad Stufe 2 entschieden (A21)".

---

## 5. Fertiger Ersatztext für §2.12a

Der bestehende §2.12a ist richtig, aber an genau den drei Punkten ungenau, die im Feld
wehtun: Einmal-URL, Familien-Sperre, Reichweite. Ersetze ihn durch:

> ### 2.12a ESP-Hub: Programmieren & OTA — Pflicht `[FIX]`
>
> Vollständig: [HUB-INTEGRATION.md](HUB-INTEGRATION.md) · verifizierter Contract:
> [AUFTRAG-CURSOR-3.md](AUFTRAG-CURSOR-3.md) Kapitel 1 · Entscheidung **A9**.
> Referenz: `iobroker.esp-hub` **v0.5.8**. Es sind **keine** Adapteränderungen nötig.
> `Schnittstellen.md` (v0.1.0) ist unvollständig und nicht die Quelle.
>
> **Muss — Inventar:**
> - `POST /api/register` alle `interval` s (Default 30) mit `mac`, `name`,
>   `hwType:"esp32"`, `chipModel`, `version` (SemVer), `ip`, `rssi`, `uptime`,
>   `freeHeap`, `freeSketch`, `fwType:"bt-gateway"` und `ios` mit Gateway-KPIs
>   inklusive `otaState`.
> - `chipModel` ist **sicherheitsrelevant**: die Chip-Familien-Sperre des Hubs erlaubt
>   den Flash, sobald eine Seite unbekannt ist. Ohne das Feld ist ein S3-Image auf
>   diesem Gerät nicht mehr blockiert.
> - Der Gerätename wird nur beim **ersten** Heartbeat übernommen (A15/A20).
>
> **Muss — Partitionen & Budget:**
> - Dual-OTA ohne Factory-Slot, App-Slots je `0x1E0000`, Tabelle nach Abschnitt D.
> - App-Image **< 1 966 080 B**; Reserve pro Release dokumentiert (R25).
> - WebUI-Assets in die App eingebettet; kein Asset-Dateisystem.
>
> **Muss — OTA:**
> - Pull über HTTP aus der `otaUrl` der Heartbeat-Antwort.
> - Die `otaUrl` wird vom Hub **nur einmal** ausgeliefert → sofortige Persistenz in NVS,
>   eigenständiges Wiederaufsetzen nach Defer, Fehler oder Reset.
> - OTA **nicht** während `STREAMING`; der Wartezustand ist sichtbar (`ios`, WebUI).
> - App-Descriptor-Prüfung (Magicword `0xABCD5432` an Offset `0x20`) und Größenprüfung
>   **vor** dem ersten Schreiben in die OTA-Partition.
> - Rollback aktiv; Freigabe der neuen App erst nach erfolgreichem Heartbeat.
> - Backoff und Abbruch nach 5 Versuchen; kein Dauer-Download während der Fahrt.
>
> **Muss — Update-Pfade (zwei Stufen, A21):**
> - Stufe 1 Heim-WLAN: Hub-OTA wie oben.
> - Stufe 2 Fahrzeug: lokales `POST /ota-upload` am Gateway (PSK, gleiche Prüfungen),
>   bedient von PiDrive aus einem lokalen Firmware-Depot. Begründung: Der Hub ist im
>   Fahrzeug nicht erreichbar und bettet seine Heim-IP in die OTA-URL ein (R24).
>
> **Muss — Release:**
> - `bt-gateway.<semver>.usb.esp32.bin` (Merged, USB `0x0`) **und**
>   `bt-gateway.<semver>.esp32.bin` (App-only, OTA).
> - Nie `.esp32s3.bin`.
> - Erstflash per Merged-Image löscht NVS: WLAN, Hub-Host, PSK **und BT-Bonding zum
>   BMW** — erneutes Pairing nötig (§2.22).
>
> **Muss nicht:** Hub-Compile-Tab (arduino-cli), Arduino-`esp-hub-base` als Basis,
> Einzel-States pro KPI im ioBroker (Q9).
>
> **Darf nicht:** OTA in aktiver A2DP-Session ohne Override-Policy; Dauer-Retry des
> Downloads; **Verwerfen einer empfangenen `otaUrl` ohne Persistenz.**

---

## 6. Arbeitspakete (ersetzt `HUB-INTEGRATION.md` §5)

Alles hier ist **Planung und Messung**, keine Firmware. Die Pakete mit Bau-Anteil sind
als „nach A17" markiert und respektieren damit `PHASE-0-MESSPLAN.md` §1.

| # | Paket | Wann | Ersetzt |
|---|-------|------|---------|
| **H1** | Partitionstabelle 2.1 in §2.12a + Abschnitt D festschreiben (Spezifikation, keine Datei) | **jetzt** | H1 umgedeutet |
| **H2** | `docs/planung/FLASH-BUDGET.md`: Messmethode 2.2 + Zahlen aus den Upstream-Beispielen | **jetzt — Input für A17** | **neu** |
| **H3** | A17-Matrix um Spalte „Codegröße" erweitern; A9 auf „bestätigt"; A21/R24/R25 aufnehmen | **jetzt** | **neu** |
| **H4** | §2.12a durch Kapitel 5 ersetzen; Pflichtenheft auf V2.0 mit Marken; `HUB-INTEGRATION.md` an H-F1–H-F9 angleichen | **jetzt** | — |
| **H5** | `PIDRIVE-INTEGRATION.md`: `/ota-upload`-Contract + Firmware-Depot (nur Spezifikation) | **jetzt** | **neu** |
| **H6** | `components/hub_client/`: Register + Antwortauswertung + `ios` | nach A17 | H2 |
| **H7** | `components/ota_manager/`: NVS-Persistenz, Defer, App-Desc-Prüfung, Rollback-Freigabe, Backoff | nach A17, **vor** dem ersten Feld-OTA | **neu, aus H-F2** |
| **H8** | SoftAP-Setup + NVS-Config (WLAN, Hub, PSK, BT-Name) + WLAN-Logzugang | nach A17, mit WebUI | H3 erweitert |
| **H9** | `ota_allowed()` an die State Machine koppeln; lokales `POST /ota-upload` | nach A17, vor dem ersten Feldgerät | H4 + H6 hochgezogen |
| **H10** | `docs/RELEASE.md` mit Checkliste 2.5 | vor dem ersten Feldgerät | H5 |

**Reihenfolge.** H1–H5 sind reine Dokumentenarbeit und können **sofort** laufen, parallel
zu Phase −1 — H2 liefert dabei eine Eingangsgröße für A17, blockiert also nichts,
sondern beschleunigt. H6–H10 warten auf A17. In Phase 0 im Labor bleibt `idf.py flash`
zulässig; die Hub-Strecke muss stehen, **bevor** ein Gerät dauerhaft im Auto verbaut
wird.

---

## 7. Pfade nach dem PiDrive-Dokumentenumzug nachziehen

*(Übernommen aus Anweisung 2, Abschnitt 4.7 — fehlt im hochgeladenen Commit `b11cffe`
und damit im Repo. Weiterhin offen: `PIDRIVE-INTEGRATION.md` verweist unverändert auf
die alten Wurzelpfade.)*

Im `pidrive`-Repo wandern alle Dokumente aus dem Wurzelverzeichnis nach `docs/`
(dortiges Arbeitspaket D1–D3). Betroffen sind genau die Dateien, die von hier aus
verlinkt werden:

- `PIDRIVE-INTEGRATION.md` Abschnitt 2 (Kasten „Docs (Pflichtlektüre für Gateway)") und
  Abschnitt 11 („Top-Dateien für Implementierer") verweisen auf `iDriveBt.md`,
  `BluetoothError.md`, `ARCHITECTURE.md`, `RUNTIME_FLOWS.md`.
- Anweisung 2 verweist in Kapitel 2, 3 und 4.2 auf `pidrive/iDriveBt.md`,
  `pidrive/RUNTIME_FLOWS.md`, `pidrive/AUFTRAG-MENUE-UND-GATEWAY.md`.
- Neu hinzugekommen: `AVRCP-MOEGLICHKEITEN.md` Kopf und §6 sowie
  `PHASE-0-MESSPLAN.md` §2.2 verweisen ebenfalls auf `pidrive`-Wurzelpfade.

**Vorgehen:** Nicht raten und nicht im `pidrive`-Repo editieren. Die PiDrive-Seite
liefert eine **Pfad-Mapping-Tabelle** (alt → neu); erst dann die Verweise hier in
**einem** Commit nachziehen. Bis die Tabelle vorliegt, bleiben die alten Pfade stehen —
ein falsch geratener Pfad ist schlechter als ein bekannt veralteter.

Grobe Erwartung, damit die Planung damit rechnen kann:
`pidrive/docs/fahrzeug/iDriveBt.md`, `pidrive/docs/betrieb/BluetoothError.md`,
`pidrive/docs/architektur/ARCHITECTURE.md`, `pidrive/docs/architektur/RUNTIME_FLOWS.md`,
`pidrive/docs/auftraege/AUFTRAG-MENUE-UND-GATEWAY.md`. Eine mögliche zweite Stufe
benennt die Dateien zusätzlich um (dort Entscheidung E7) — deshalb **einmal** nachziehen,
wenn beide Stufen durch sind, nicht zweimal.

---

## 8. Was ausdrücklich nicht zu tun ist

1. **Keinen Adaptercode in `iobroker.esp-hub` ändern.** Alles hier funktioniert mit
   v0.5.8 wie ausgeliefert. Änderungswünsche → Q9.
2. **Nicht im `pidrive`-Repo arbeiten.** Firmware-Depot und Probe-Skripte werden hier
   nur spezifiziert bzw. verlinkt.
3. **Keine Firmware anlegen.** Anweisung 2 §5 und `PHASE-0-MESSPLAN.md` §1 gelten
   unverändert: kein `main/`, kein `components/` vor A17. Die Flash-Messung (2.2) läuft
   mit **Upstream-Beispielen außerhalb des Repos** — das ist der Grund, warum sie
   trotzdem sofort möglich ist.
4. **Keine Stack-Entscheidung (A17)** ohne Phase −1 **und** ohne die Größenmessung.
5. **`Schnittstellen.md` nicht als Wahrheit zitieren.** Sie ist v0.1.0 und kennt
   `chipModel`, `freeSketch`, `/api/ota-push` und das Löschverhalten der `otaUrl` nicht.
   Immer auf `main.js` mit Zeilennummer verweisen.
6. **Die Partitionstabelle nicht später „aufräumen".** Jede Änderung an den Offsets
   kostet im Feld einen USB-Flash und damit die Kabellosigkeit.
7. **`0x1E0000` nicht vergrößern**, um ein zu großes App-Image unterzubringen. Das ist
   die Stelle, an der still das Dual-OTA stirbt. Stattdessen R25 eskalieren.
8. **Keine zweite Freigabe-Checkliste** neben `OFFENE-PUNKTE.md` Abschnitt E anlegen.

---

## 9. Entscheidungen für den Owner (Q6–Q11)

Q1–Q5 aus Anweisung 2 sind weiterhin **unbeantwortet** und blockieren A17/A18.
Zusätzlich:

| # | Frage | Bezug | Warum sie jetzt kommt |
|---|-------|-------|----------------------|
| **Q6** | Referenzboard: D1 Mini ESP32 (4 MB, Hub-Standard) oder WROVER mit PSRAM (besser für Jitter-Buffer und Menübaum)? Und: falls die Messung eng wird — ist ein 8/16-MB-Modul akzeptabel, obwohl es von der Hub-Referenzhardware abweicht? | **A16**, R25 | Bestimmt, ob die Tabelle aus 2.1 trägt oder neu gerechnet wird |
| **Q7** | Soll ein laufender **Analyzer-Lauf** OTA blockieren wie `STREAMING`, oder nur der Stream? | 2.3, §2.5 | Definiert `ota_allowed()` |
| **Q8** | `freeSketch`-Semantik: freie Slot-Größe wie bei Arduino (Balken ~100 %) oder echte Reserve `Slot − App` (Balken zeigt das Budget)? Letzteres weicht von der Familie ab, ist aber die Zahl, die in 2.2 zählt | H-F7 | Konsistenz mit den anderen `esp32.*`-Geräten |
| **Q9** | Sollen die Gateway-KPIs **einzelne** ioBroker-States bekommen (für VIS/Skripte) statt eines `ios`-JSON-Strings? Das wäre eine Adaptererweiterung | H-F8 | Heute nicht nötig — aber eine bewusste Entscheidung statt eines Versäumnisses |
| **Q10** | Firmware-Signatur (`CONFIG_SECURE_SIGNED_APPS_NO_SECURE_BOOT`) für den OTA-Pfad, oder genügen App-Desc-Prüfung + Rollback? Der Hub liefert ohne TLS und ohne Auth | H-F1, H-F9 | Aufwand gegen Schutz vor falschem/manipuliertem Image |
| **Q11** | Update-Pfad Stufe 2 (PiDrive-Depot) verbindlich, oder im Auto bewusst USB akzeptieren? | **A21**, R24 | Entscheidet, ob „kabellos" auch im Fahrzeug gilt |

---

## 10. Definition of Done für diese Anweisung

- [ ] Partitionstabelle 2.1 in §2.12a und `OFFENE-PUNKTE.md` Abschnitt D festgeschrieben
- [ ] `docs/planung/FLASH-BUDGET.md` mit Messmethode und ersten Zahlen (Bluedroid vs. BTstack)
- [ ] A17-Matrix um „Codegröße" erweitert; A9 auf „bestätigt" mit Verweis auf Kapitel 1
- [ ] A21, R24, R25 in `OFFENE-PUNKTE.md`; Abschnitt E um die vier Punkte aus 4.3 ergänzt
- [ ] Pflichtenheft auf **V2.0**, eine Datei, `[FIX]`/`[ENTWURF — Gate: …]` pro Kapitel,
      Titel/Status-Widerspruch aufgelöst, Änderungshistorie am Ende
- [ ] §2.12a durch den Text aus Kapitel 5 ersetzt
- [ ] §2.12 um „WebUI eingebettet, kein Asset-Dateisystem" ergänzt; §2.22 um den
      NVS-/Bonding-Verlust beim Merged-Flash
- [ ] `HUB-INTEGRATION.md` §1/§5/§6 an H-F1–H-F9 angeglichen; §7 zitiert `main.js`-Zeilen
- [ ] `PIDRIVE-INTEGRATION.md`: `/ota-upload`-Contract + Firmware-Depot ergänzt
- [ ] `REVIEW-V1.3.md` mit den Befunden H-F1–H-F9 analog zu `REVIEW-V1.2.md`
- [ ] `STATE.md`/`README.md`: „Planung V2.0 — Teil A verbindlich"
- [ ] Kein `main/`, kein `components/`, keine Änderung in `iobroker.esp-hub` und `pidrive`
