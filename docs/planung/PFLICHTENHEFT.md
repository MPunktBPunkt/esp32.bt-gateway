# Pflichtenheft V2.0 — PiDrive Bluetooth Gateway (ESP32)

**Dokumentstatus:** Teil A verbindlich `[FIX]`, Teil B Entwurf `[ENTWURF — Gate: …]`  
**Ziel:** Vollständige, implementierbare Spezifikation für ein eigenständiges ESP-IDF-Projekt + minimale PiDrive-Erweiterung.  
**Review-Nachzüge:** [REVIEW-V1.1.md](REVIEW-V1.1.md), [REVIEW-V1.2.md](REVIEW-V1.2.md), [REVIEW-V1.3.md](REVIEW-V1.3.md)  
**Messplan:** [PHASE-0-MESSPLAN.md](PHASE-0-MESSPLAN.md) · **AVRCP:** [AVRCP-MOEGLICHKEITEN.md](AVRCP-MOEGLICHKEITEN.md) · **Flash:** [FLASH-BUDGET.md](FLASH-BUDGET.md)  
**Hub-Contract:** [AUFTRAG-CURSOR-3.md](AUFTRAG-CURSOR-3.md) Kap. 1 · [HUB-INTEGRATION.md](HUB-INTEGRATION.md)

Marken: Jedes Kapitel trägt im ersten Absatz `[FIX]` oder `[ENTWURF — Gate: …]`. Freigabe in [OFFENE-PUNKTE.md](OFFENE-PUNKTE.md) Abschnitt E — keine zweite Checkliste hier.

---

## 2.1 Systemarchitektur & Verantwortungsgrenzen

`[FIX]` — durch PiDrive-Analyse bestätigt, in V1.2 nachgezogen.

### 2.1.1 Komponenten

| Komponente | Verantwortung | Kennt nicht |
|------------|---------------|-------------|
| **PiDrive** | Quellen, Audio-Engine, Trigger-Dispatcher, Metadaten-Erzeugung, Gateway-Client, Routing-Logik | A2DP, SBC, AVRCP, BlueZ (Gateway-Pfad) |
| **ESP32 Gateway** | WiFi, PDAP, Jitter-Buffer, SBC-Encoder, A2DP Source, AVRCP Target (+ optional Browsing), Connection-Management, Analyzer, WebUI/Diagnose, **ESP-Hub Heartbeat/OTA**, lokales Feld-OTA | DAB, Spotify, PiDrive-Menüsemantik (nur generische Items bei S3) |
| **iobroker.esp-hub** | Geräteinventar, USB-Flash, Firmware-Ablage, OTA-URL in Heartbeat | Audio, BT, PDAP, BMW |
| **BMW** | A2DP Sink + AVRCP Controller | Herkunft des Audios |

### 2.1.2 Datenflüsse

**Audio (Pi → ESP → BMW):**
```
Quelle → PipeWire → Capture-Node (erzwungen 44,1 kHz / S16LE / Stereo)
→ PDAP Audio-Kanal → ESP Jitter-Buffer → SBC → A2DP → BMW
```

Hinweis: Ohne Capture-Node wären Quellformate gemischt (u. a. FM 32 kHz mono). Der Pi resample’t; der ESP bleibt format-fest.
**Steuerung (BMW → ESP → Pi):**
```
BMW AVRCP → ESP Analyzer → PDAP Event (next|previous|play_pause|…)
→ PiDrive map_event() → /tmp/pidrive_cmd → Trigger-Dispatcher
```

**Metadaten (Pi → ESP → BMW):**
```
PiDrive Metadata → PDAP → ESP → AVRCP Metadata → BMW Display
```

**Status (ESP → Pi):**
```
ESP Status (WiFi, BT, Buffer, KPIs) → PDAP → PiDrive
```

### 2.1.3 Schichtenmodell

- **Layer 1 – Transport:** WiFi (TCP/UDP)
- **Layer 2 – PDAP:** PiDrive Gateway Protocol
- **Layer 3 – Bluetooth Classic:** A2DP Source + AVRCP Target

---

## 2.2 Explizite Nicht-Ziele (V1)

`[FIX]` — F4 belegt (kein Sink+Source); S3-Ausschluss technisch zwingend.

- **A2DP-Relay nicht im ESP:** Sink + Source gleichzeitig auf dem Gateway entfällt dauerhaft (Espressif: „A2DP source cannot be used together with A2DP sink at the same time“ — F4). Multi-Source (Handy, Tablet) läuft über den **Pi als A2DP-Sink** → PipeWire → PDAP → ESP → BMW (Weg F, [BETRIEBSMODI.md](BETRIEBSMODI.md); Entscheidung A19). Der ESP bleibt Ein-Rollen-Gerät (Source + Target).
- Mehrere gleichzeitige A2DP-Sinks am ESP
- Mehrere gleichzeitige PDAP-Audio-Sessions (ein Session-Owner)
- AAC, aptX, LDAC
- HFP / Telefonie
- Audio-Mixing oder Quellen-Umschaltung im ESP
- BMW-spezifische Geschäftslogik im ESP (Semantik der Menübaum-Knoten bleibt beim Client — §2.24)
- ESP32-S3 oder andere BLE-only-Chips
- Vollständige BlueZ-Emulation
- Automatisches Resampling oder Clock-Drift-Korrektur (nur Überwachung)
- Dauerhaftes APSTA+A2DP ohne bestandenes Gate (siehe Betriebsmodi)

**Erlaubt / geplant (nicht Nicht-Ziel):** Handy-Musik **über PiDrive** (Spotify Connect, Pi-A2DP-Sink, …) oder später als **PDAP-Client** (WLAN/SoftAP). Details: [BETRIEBSMODI.md](BETRIEBSMODI.md).

---

## 2.3 Phase −1 / Phase 0 – Vermessung vor Implementierung

`[ENTWURF — Gate: Phase −1 abgeschlossen]` · Ergebnisort: [PHASE-0-MESSPLAN.md](PHASE-0-MESSPLAN.md), A17/A18.

Vollständig: [PHASE-0-MESSPLAN.md](PHASE-0-MESSPLAN.md).

**Phase −1 (Pi, vor ESP-Firmware):** Mit BlueZ/`btmon` am realen BMW klären: Browsing-Kanal? Metadata-Zeilen? Pass-Through-Subset? Blockiert A17 (Host-Stack) und A18 (Menü). Durchführung in `pidrive` (G1/G2).

**Phase 0 (ESP-Analyzer):** Discovery/Pairing/Connect; vollständiges Logging eingehender AVRCP-PDUs, Timing, Codec-Negotiation, Disconnect-Reasons; WebUI + exportierbare Logs; Self-Test-Testton.

**Exit Phase −1:** Entscheidungswirkung dokumentiert (S3 ja/nein → Stack-Empfehlung).  
**Exit Phase 0:** Mapping-Tabelle aus realem Verhalten über Zündzyklen — nicht spekulativ vorab.

---

## 2.4 WiFi + Classic-BT Coexistence (Gate-Kriterium)

`[FIX]` — Kriterien stehen; nur die Durchführung ist stackabhängig (nach A17).

**Anforderung:**  
Der klassische ESP32 muss gleichzeitig WiFi (WebUI + Heartbeat + PDAP-Traffic) und stabile A2DP-Verbindung zum BMW halten.

**Gate-Tests (vor jeder weiteren Arbeit):**

1. **STA + A2DP** ≥ 30 min (Heim/Carport-Muster)  
2. **SoftAP + A2DP** ≥ 30 min (Auto ohne Heim-WLAN / Handy als PDAP-Client)  
3. Optional: **APSTA + A2DP** nur wenn Betriebsmodus APSTA gewünscht

Jeweils: keine hörbaren Dropouts, kein unerwarteter Disconnect; KPIs Packet-Loss, Jitter, Buffer-Underruns, RSSI, CPU.

**Bei Nichtbestehen:** Architekturänderung (z. B. externes WiFi-Modul oder anderer Ansatz) vor Fortsetzung. SoftAP-only-Setup ohne Stream kann trotzdem für Erstkonfiguration bleiben.

WiFi-Policy (STA / SoftAP / APSTA): [BETRIEBSMODI.md](BETRIEBSMODI.md).

---

## 2.5 ESP-Zustandsautomat

`[FIX]` — unabhängig vom BT-Stack; Sonderzustand `OTA_PENDING` siehe §2.12a.

**Hauptzustände:**
```
BOOT
→ WIFI_CONNECTING
→ WIFI_CONNECTED
→ BMW_DISCOVERY
→ BMW_PAIRING
→ BMW_CONNECTING
→ BMW_CONNECTED
→ A2DP_READY
→ STREAMING
```

**Sonderzustände:**

- `NO_BMW` (normal, kein Fehler)
- `RECONNECTING` (mit Backoff)
- `ERROR_xxx`
- `BUFFERING` / `RECOVER`
- `OTA_PENDING` (otaUrl in NVS, Warte auf `ota_allowed`)

**Backoff-Strategie bei Disconnect:** 1 s → 2 s → 5 s → 10 s → 30 s → 60 s

**Regel:** „BMW disconnected“ (Zündung aus) ist ein erwarteter Zustand und löst keinen Fehleralarm aus.

---

## 2.6 A2DP-Audioarchitektur

`[ENTWURF — Gate: Coexistence-Gate + Hörtest]` · Ergebnisort: KPI-Logs / Bitpool-Defaults in WebUI.

- Rolle: **A2DP Source**
- Codec: **SBC only**
- Sample-Rate: fest **44,1 kHz** (Pi-Capture erzwingt das; ESP akzeptiert in V1 nur dieses Format)
- Format: **16 Bit, Stereo, S16LE**
- Bitpool: parametrisierbar (Startwert 32–35), empirisch optimieren
- Nur **eine** gleichzeitige A2DP-Sink-Verbindung (BMW)

**SBC-Encoding** läuft auf dem ESP. Der Pi liefert ausschließlich PCM im Contract-Format.  
BMW/BlueZ-Erfahrung in PiDrive: SBC only, Absolute Volume vermeiden (Lautstärke = BMW-DSP + Lenkrad-Events).

**Sample-Rate-Konflikt:** A2DP kann 44,1 oder 48 kHz verhandeln. V1 zuerst **SBC auf 44,1 erzwingen**; wenn der BMW nur 48 akzeptiert → ESP resample’t PCM 44,1→48 vor SBC (CPU messen). Kein stilles Mismatch.

---

## 2.7 Jitter-Buffer & Audio-Clock

`[ENTWURF — Gate: Coexistence-Gate + Hörtest]` · Ergebnisort: Buffer-Defaults nach Laborfahrt.

- Buffer-Größe: parametrisierbar (20 / 40 / 60 / 80 / 100 / 150 / 200 ms)
- WebUI zeigt aktuellen Füllstand, Target, Underruns, Overruns, Packet-Loss, Jitter
- Bei Packet-Loss: Silence-Frame oder letzte Frame wiederholen (kein Knacken)
- **Audio-Clock-Drift:** In V1 nur überwachen und anzeigen. Keine Korrektur (Resampling/Frame-Drop). Architektur muss spätere Korrektur erlauben.

---

## 2.8 PDAP – PiDrive Gateway Protocol

`[FIX]` für Kanäle/Rollen **ohne** Byte-Layout.  
`[ENTWURF — Gate: Audio-Strecke im Labor]` für Byte-Layout, CRC, Feldbreiten · Ergebnisort: künftiges `PDAP.md`.

**Logische Kanäle (transportunabhängig):**

**Control-Kanal** (reliable, ordered – V1: TCP oder WebSocket):

- HELLO / AUTH
- STATUS
- COMMAND
- EVENT
- METADATA
- HEARTBEAT
- CONFIG
- ERROR

**Audio-Kanal** (verlusttolerant – V1: UDP):

- AUDIO_START
- AUDIO_DATA
- AUDIO_STOP
- AUDIO_FLUSH

**Minimaler Audio-Header (Vorschlag):**
```
MAGIC | VERSION | TYPE | STREAM_ID | SEQ | TIMESTAMP_MS
SAMPLE_RATE | CHANNELS | FORMAT | FLAGS | PAYLOAD_LEN | CRC16
+ PAYLOAD
```

**Heartbeat:** alle 1–2 s in beide Richtungen. Nach 3 fehlenden Heartbeats → Gateway offline.

**HELLO / Capabilities (V1):** neben AUTH/VERSION meldet der ESP u. a. `fw_version`, `bt_name`, `capabilities` (z. B. `audio_pcm_44100_s16le`, `metadata_v1`, `events_v1`, optional `menu_v1`), `max_buffer_ms`. Client bricht bei Inkompatibilität klar ab.

**Security (V1 minimal):** Pre-Shared-Key / Token im HELLO/AUTH.

**Discovery (Pi findet ESP):** Config `gateway_host` auf dem Pi (Pflicht-Minimum); empfohlen mDNS `bt-gateway.local` im STA-LAN; SoftAP-SSID-Muster für Setup/Auto. Details: [REVIEW-V1.1.md](REVIEW-V1.1.md) L1.

**Optionaler Menü-Kanal (bei Ausbaustufe S3):** Control-Nachrichten zur generischen Item-Liste (Skizze §2.24 / Auftrag §4.6) — `MENU_TREE`, `MENU_INVALIDATE`, `MENU_ACTIVATE`, optional `MENU_PAGE_*`. Byte-genaue Structs bewusst offen.

---

## 2.9 AVRCP

`[ENTWURF — Gate: Phase-0-Messung am realen NBT]` · Ergebnisort: Mapping-Tabelle nach Phase 0.

- ESP = **AVRCP Target** (+ bei S3 Browsing-Target)
- BMW = **AVRCP Controller** (+ ggf. Browsing-Controller)
- V1: Vollständiger Analyzer (alles loggen)
- Später: Mapping der beobachteten Commands auf PDAP-Events mit **PiDrive-Event-Namen**:
  `next`, `previous`, `play`, `pause`, `play_pause`, `stop`, `volumeup`, `volumedown`, `fast_forward`, `rewind`
- Keine Geschäftslogik im ESP – der Pi mappt via bestehendem `map_event()` (Menü vs. FM vs. DAB)
- Double-Tap-Semantik (1,2 s → `cat:0`) bleibt auf dem Pi (S1); bei S3 weitgehend überflüssig
- Absolute Volume: nicht gegen BMW kämpfen (PiDrive-Praxis: Volume fix / DSP im Auto)

**Ausbaustufe S3 — Browsing** (nur wenn Phase −1 positiv; Details [AVRCP-MOEGLICHKEITEN.md](AVRCP-MOEGLICHKEITEN.md)):

Target-seitig u. a. zu unterstützen: `SetBrowsedPlayer`, `ChangePath`, `GetFolderItems`, `GetItemAttributes`, `GetTotalNumberOfItems`, `PlayItem` (optional Search). Voraussetzung: Host-Stack mit Browsing-API (**A17**, typisch BTstack). PDAP-Menü-Kanal §2.24.

---

## 2.10 Metadata-Engine

`[ENTWURF — Gate: Phase −1 (Zeilen?) + A17]` · Ergebnisort: A17 + Phase-0-Attribute-Log.

**Umsetzbarkeit ist stackabhängig (F1):** Die öffentliche Bluedroid-Target-API kann keine Element-Attributes setzen (`GetElementAttributes` wird nicht an die App gereicht; `esp_avrc_ct_send_metadata_cmd` gilt nur für die Controller-Rolle). Ohne BTstack oder IDF-Patch bleibt das BMW-Display im Gateway-Pfad leer — Entscheidung **A17** ([OFFENE-PUNKTE.md](OFFENE-PUNKTE.md)).

- Pi sendet strukturierte Metadata analog zu den heutigen MPRIS-Feldern (Title, Artist, Album, Source, Playing-Status …)
- Felder stammen aus denselben Inputs wie `mpris2.update()` / Playback-Status / DLS / ICY
- ESP setzt daraus AVRCP TrackChanged + Element Attributes (**sofern der gewählte Stack das erlaubt**)
- Zuerst observe (welche Attribute der BMW anfordert), dann implementieren
- Bei aktivem Gateway ist BlueZ-MPRIS fürs BMW-Display optional idle
- Bei S3: die drei Zeilen bleiben parallel sinnvoll (Now Playing); die Liste kommt über Browsing

---

## 2.11 Control API & Statusmodell

`[FIX]` — unabhängig vom BT-Stack.

Der Pi behandelt den ESP wie ein Netzwerkgerät.

Mindest-Status:

- WiFi (RSSI, verbunden)
- Bluetooth (State, RSSI, Codec, Bitpool, MTU, Reconnect-Count, Disconnect-Reason)
- Audio (Streaming, Buffer-Level, Underruns/Overruns)
- Source / Track (von Pi gespiegelt)
- OTA-Zustand (`idle` / `pending` / `downloading` / `error`) — Spiegel in Hub-`ios`

---

## 2.12 WebUI

`[FIX]` für Einbettungsregel und Funktionsumfang (ohne pixelgenaue UX).

Funktionen:

- Live-Status (BMW, A2DP, AVRCP, Buffer, WiFi, `otaState`)
- AVRCP-Analyzer-Anzeige
- Self-Test (Testton)
- Buffer- und SBC-Parameter
- Logs / Export (RAM-Ringpuffer + Systemlog über WLAN — Fehlersuche ohne USB)
- Verbindung manuell steuern (Connect / Disconnect / Reconnect)
- Hub-Host/Port konfigurieren; **lokales OTA-Upload** (`POST /ota-upload`, Stufe 2 — Pflicht, A21)

**Einbettung:** WebUI-Assets werden **in die App eingebettet** (`EMBED_FILES` / `EMBED_TXTFILES`). Kein Asset-Dateisystem, keine eingebetteten Fonts, keine Chart-Bibliothek als Datei. Diagramme falls nötig: Inline-SVG wie `esp-hub-base`. Begründung: Flash-Budget ([FLASH-BUDGET.md](FLASH-BUDGET.md)).

---

## 2.12a ESP-Hub: Programmieren & OTA — Pflicht

`[FIX]` — vollständig gegen Adaptercode verifiziert (A9).  
Hinweis: Die Hub-Schnittstelle ist fix; der genaue Zeitpunkt von `ota_allowed()` hängt an der State Machine (Teil B) — das ist beabsichtigt, kein Widerspruch.

Vollständig: [HUB-INTEGRATION.md](HUB-INTEGRATION.md) · verifizierter Contract:
[AUFTRAG-CURSOR-3.md](AUFTRAG-CURSOR-3.md) Kapitel 1 · Entscheidung **A9**.
Referenz: `iobroker.esp-hub` **v0.5.12** (lokal verifiziert; Auftrag bezog sich auf v0.5.8 — Kernverhalten OTA/Flash/chipModel unverändert). Es sind **keine** Adapteränderungen nötig.
`Schnittstellen.md` (v0.1.0) ist unvollständig und nicht die Quelle.

**Muss — Inventar:**
- `POST /api/register` alle `interval` s (Default 30) mit `mac`, `name`,
  `hwType:"esp32"`, `chipModel`, `version` (SemVer), `ip`, `rssi`, `uptime`,
  `freeHeap`, `freeSketch`, `fwType:"bt-gateway"` und `ios` mit Gateway-KPIs
  inklusive `otaState`.
- `chipModel` ist **sicherheitsrelevant**: die Chip-Familien-Sperre des Hubs erlaubt
  den Flash, sobald eine Seite unbekannt ist. Ohne das Feld ist ein S3-Image auf
  diesem Gerät nicht mehr blockiert.
- Der Gerätename sollte bei der **Erstregistrierung** stimmen (A15/A20). Ab Adapter
  v0.5.12 wird ein nicht-leeres `name` bei jedem Heartbeat übernommen (Korrektur zu H-F4
  aus dem Auftrag — siehe [REVIEW-V1.3.md](REVIEW-V1.3.md)).

**Muss — Partitionen & Budget:**
- Dual-OTA ohne Factory-Slot, App-Slots je `0x1E0000`, Tabelle nach [OFFENE-PUNKTE.md](OFFENE-PUNKTE.md) Abschnitt D / [FLASH-BUDGET.md](FLASH-BUDGET.md):

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

- App-Image **&lt; 1 966 080 B**; Reserve pro Release dokumentiert (R25).
- WebUI-Assets in die App eingebettet; kein Asset-Dateisystem.

**Muss — OTA:**
- Pull über HTTP aus der `otaUrl` der Heartbeat-Antwort.
- Die `otaUrl` wird vom Hub **nur einmal** ausgeliefert → sofortige Persistenz in NVS
  (`pending_ota_url`, `pending_ota_since`, `pending_ota_attempts`), eigenständiges
  Wiederaufsetzen nach Defer, Fehler oder Reset.
- OTA **nicht** während `STREAMING`; der Wartezustand ist sichtbar (`ios`, WebUI).
  Analyzer-Lauf: siehe Q7.
- App-Descriptor-Prüfung (Magicword `0xABCD5432` an Offset `0x20`) und Größenprüfung
  **vor** dem ersten Schreiben in die OTA-Partition.
- Rollback aktiv (`CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE=y`); Freigabe der neuen App
  erst nach erfolgreichem Heartbeat (`esp_ota_mark_app_valid_cancel_rollback`).
- Backoff und Abbruch nach 5 Versuchen; kein Dauer-Download während der Fahrt.

**Muss — Update-Pfade (zwei Stufen, A21):**
- Stufe 1 Heim-WLAN: Hub-OTA wie oben.
- Stufe 2 Fahrzeug: lokales `POST /ota-upload` am Gateway (PSK, gleiche Prüfungen),
  bedient von PiDrive aus einem lokalen Firmware-Depot. Begründung: Der Hub ist im
  Fahrzeug nicht erreichbar und bettet seine Heim-IP in die OTA-URL ein (R24).
  Contract: [PIDRIVE-INTEGRATION.md](PIDRIVE-INTEGRATION.md).

**Muss — Release:**
- `bt-gateway.<semver>.usb.esp32.bin` (Merged, USB `0x0`) **und**
  `bt-gateway.<semver>.esp32.bin` (App-only, OTA).
- Nie `.esp32s3.bin`.
- Erstflash per Merged-Image löscht NVS: WLAN, Hub-Host, PSK **und BT-Bonding zum
  BMW** — erneutes Pairing nötig (§2.22).

**Muss nicht:** Hub-Compile-Tab (arduino-cli), Arduino-`esp-hub-base` als Basis,
Einzel-States pro KPI im ioBroker (Q9).

**Darf nicht:** OTA in aktiver A2DP-Session ohne Override-Policy; Dauer-Retry des
Downloads; **Verwerfen einer empfangenen `otaUrl` ohne Persistenz.**

---

## 2.13 Diagnose, Logging & KPIs

`[FIX]` — unabhängig vom BT-Stack.

Mindest-KPIs (WebUI + API + Log + Hub-`ios`):

- WiFi: RSSI, Packet-Loss, Jitter
- Buffer: Level, Underruns, Overruns
- SBC: Frames, Drop-Rate, CPU
- BT: States, Codec, Bitpool, MTU, RSSI, Disconnect-Reasons
- High-Level: Source, Track, Playing
- OTA: `otaState` (idle / pending / …)

Empfohlener `ios`-Ausschnitt (klein, gleiche Funkstrecke wie A2DP):

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

---

## 2.14 Recovery & Fehlerkonzept

`[FIX]`.

- WiFi-Verlust → Gateway meldet `GATEWAY_AUDIO_LOST` → PiDrive entscheidet über Fallback (AUX/HDMI/Stop)
- BMW-Disconnect → Backoff + Reconnect-Versuch
- Buffer-Underrun → Silence + Status
- ESP-Reset → möglichst schneller Wiederaufbau der Bluetooth-Seite; pending OTA aus NVS
- Kaputtes OTA-Image → Rollback (Bootloader), bis Heartbeat-Validierung

---

## 2.15 Security (V1)

`[FIX]`.

- Pre-Shared-Key / Token zwischen Pi und ESP (PDAP + lokales `/ota-upload`)
- Keine offenen Commands ohne Authentifizierung
- Hub-Register ohne Auth: reines Heimnetz; darüber **keine** sicherheitsrelevanten Kommandos annehmen (PSK gilt für PDAP/Feld-OTA, nicht für den Hub)

---

## 2.16 ESP-IDF / FreeRTOS-Architektur (V1-Skizze)

`[ENTWURF — Gate: A17]` · Ergebnisort: [OFFENE-PUNKTE.md](OFFENE-PUNKTE.md) Abschnitt D / künftiges `FREERTOS.md`.

Empfohlene Tasks (Prioritäten später festlegen):

- WiFi / Network
- Hub Client (register / OTA-gate)
- OTA Manager (NVS-Persistenz, Defer, App-Desc, Rollback)
- PDAP Control
- PDAP Audio Receiver + Jitter-Buffer
- SBC Encoder
- A2DP Source / AVRCP (Host-Stack **A17** — Bluedroid-Profile oder BTstack-Profile + ggf. `menu_target`)
- WebUI / HTTP (+ lokales OTA)
- Diagnostics / Watchdog
- State Machine

**Komponentenliste** unter `components/` erst nach A17 festschreiben ([OFFENE-PUNKTE.md](OFFENE-PUNKTE.md) Abschnitt D).

---

## 2.17 PiDrive-Integration

`[ENTWURF — Gate: A7 + PipeWire-Bridge auf dem Pi]` für DAB; Kern-Ankerpunkte sonst `[FIX]`-Richtung.

Detailliert: [PIDRIVE-INTEGRATION.md](PIDRIVE-INTEGRATION.md) (Analyse `pidrive` v0.11.127).

Mindestumfang im Repo **`pidrive`** (nicht hier):

- Neuer `integration/gateway_client.py` (+ optional `systemd/pidrive_gateway.service`)
- `audio_output = gateway` als zusätzliche Route in `modules/audio.py` / `settings.py`
- PipeWire: virtueller Sink + Resample auf Contract-PCM
- Parallelbetrieb: `audio_output = bt | gateway` (BlueZ-BMW-Pfad idle, wenn Gateway aktiv)
- Reverse: PDAP-Events → `map_event()` → `/tmp/pidrive_cmd` (Trigger-Dispatcher unverändert)
- Metadata-Push parallel zu / statt MPRIS-BMW-Pfad
- **DAB:** separates Arbeitspaket — heute Direct-ALSA; ohne PW-Bridge kein Gateway-DAB
- **Firmware-Depot + `pidrivectl gateway firmware …`** (Stufe-2-Update, A21) — Pi-Seite nachziehen

In **diesem** Repo: PDAP-Contract + `clients/pdap_tester/` (Laptop-Referenz) + `/ota-upload`-Contract.

---

## 2.18 Testplan (Labor)

`[FIX]` als Mindestrahmen.

1. ESP A2DP Source → Kopfhörer (Testton)
2. ESP + WiFi → BMW (Coexistence-Gate)
3. AVRCP-Analyzer mit realem BMW
4. Metadata-Test
5. Netzwerk-Audio (Laptop → ESP → BMW)
6. Vollständige PDAP-Strecke
7. Hub-OTA + lokales `/ota-upload` (Defer während STREAMING)

---

## 2.19 Auto-Testplan

`[FIX]` als Mindestrahmen.

- Zündungszyklen
- Lange Fahrten (30–60 min)
- Störquellen (andere WLAN/BT-Geräte)
- Schnelles NEXT/PREV
- Quellenwechsel
- WiFi-Ausfall-Simulation
- ESP-Reset während Betrieb
- Feld-Update Stufe 2 (ohne Heim-WLAN)

---

## 2.20 Exit-Kriterien je Phase

`[FIX]` als Checkliste; Messinhalte hängen an Teil B.

| Phase | Exit-Kriterium |
|-------|----------------|
| −1 | Browsing-/Metadata-Probe am Pi ausgewertet → A17/A18/S3-Wegwahl |
| Flash-Budget | Upstream-Messung Bluedroid vs. BTstack dokumentiert; Reserve ≥ 20 % oder Eskalation Q6 |
| 0 | Vollständige BMW-Protokollvermessung + Analyzer |
| Coexistence | ≥ 30 min stabil A2DP + WiFi (Stack-spezifisch) |
| A2DP | Stabiler Testton zum BMW |
| AVRCP | Alle relevanten Commands geloggt und gemappt |
| Metadata | Titel/Artist erscheinen korrekt (stackabhängig) |
| Menü | **Menü im Fahrzeug bedienbar** — bei S1: Skip/Play-Ergonomie; bei S3: Browse-Liste. Messgröße aus PiDrive: Tastendrücke bis Ziel |
| PDAP Audio | Stabiles PCM über WLAN |
| Integration | Webradio (und ggf. Spotify/lokal) über Gateway hör- und steuerbar; DAB optional nach PW-Bridge |
| Hub/OTA | Defer+NVS, Rollback, App-Desc-Prüfung, Stufe 1+2 verifiziert |
| Robustheit | Mehrere erfolgreiche Auto-Tests ohne manuellen Eingriff |

---

## 2.21 Hardware

`[FIX]` — S3 hat kein Classic BT; nicht verhandelbar.

- **Nur klassischer ESP32** (Ziel: WROOM-32 / DevKit; bei RAM-Engpass Plan B: **WROVER/PSRAM**, weiterhin kein S3)
- Flash: Referenz **4 MB** Dual-OTA (Q6: 8/16 MB nur nach Eskalation R25)
- Antenne für 2,4 GHz (Platzierung im Auto beachten)
- Stabile 5-V-Versorgung (Auto-tauglich; Verpolung/Lastabwurf hardwareseitig bedenken)
- Brownout-Detector + Task-Watchdog aktiv
- Optional später: Status-LED, Factory-Reset-Taster (V1: Reset über WebUI reicht)
- Kein ESP32-S3

---

## 2.22 Ersteinrichtung & Migration (BMW)

`[FIX]` — inkl. NVS-/Bonding-Verlust beim Merged-Flash.

1. ESP flashen (Hub-USB Merged @ `0x0` oder `idf.py`), SoftAP-Setup: WLAN, Hub-Host/Port (Default 8093), PDAP-PSK, BT-Gerätename (A15/A20)  
2. **Warnung:** Merged-Write auf `0x0` überschreibt NVS mit `0xFF` → WLAN, Hub-Host, PSK und **BT-Bonding zum BMW** sind weg; erneutes Pairing am Fahrzeug nötig. Bei Recovery gewollt; Routine-Updates nur per OTA.  
3. Am BMW altes Pi-/Dongle-Gerät entfernen oder nicht parallel verbinden  
4. ESP pairen (Name konfigurierbar, Bonding in NVS)  
5. Pi: `gateway_host` + `audio_output=gateway`; BlueZ-Auto-Connect zum BMW aus  
6. Coexistence-Gate + Webradio-Test vor DAB-Umbau  

---

## 2.23 Latenz, Logs, Reset (Kurz)

`[FIX]` für Rahmen; Persistenz optional.

- **Latenz:** Ziel Ende-zu-Ende grob &lt; 150–200 ms (Buffer parametrisierbar); Anzeige über Timestamps  
- **Logs:** RAM-Ring + WebUI-Export (WLAN-lesbar); Persistenz optional LittleFS-Partition `logs`  
- **Factory-Reset:** WebUI löscht NVS (WiFi, Bonding, Tokens); SoftAP mit WPA2-Passwort (`BT-Gateway-Setup` o. Ä.)  

---

## 2.24 Menü-Transport

`[ENTWURF — Gate: Phase −1 Browsing-Probe → S1 oder S3; dann A18]` · Ergebnisort: A18 + §2.24 Finalisierung.

Das Menü ist das zentrale Produktmerkmal der PiDrive-Bedienung am iDrive; es darf nicht nur in §2.10 „mitgemeint“ sein. Grundlage: [AVRCP-MOEGLICHKEITEN.md](AVRCP-MOEGLICHKEITEN.md).

| Stufe | Was der ESP tut | Was der Pi tut |
|-------|-----------------|----------------|
| **S1** | Metadata (3 Zeilen) + Pass-Through→Events | Menübaum, Skip=Cursor, Play=Enter, `map_event` |
| **S3** | Generische Browse-Items an BMW + `PlayItem`→PDAP | Semantik der UIDs, Baumaufbau, Invalidierung |

**Designregel:** Der ESP kennt keine PiDrive-Geschäftslogik; er transportiert eine **generische, semantikfreie** Baumstruktur (Ordner/Items/Namen/playable). Bedeutung kennt nur der Client.

**PDAP-Skizze (Diskussionsgrundlage, nicht byte-fest):**

```
MENU_TREE      { uid_counter, nodes: [ { uid, parent_uid, kind: folder|item,
                                         name, playable, attrs: {…} } ] }
MENU_INVALIDATE{ uid_counter }                      Pi → ESP: Baum neu holen
MENU_ACTIVATE  { uid }                              ESP → Pi: aus PlayItem
MENU_PAGE_REQ  { parent_uid, start, count }         optional (Lazy-Loading)
MENU_PAGE_RSP  { parent_uid, start, nodes[] }
```

**Größenbudget (Vorschlag):** ≤ 32 KB / ≤ 400 Knoten im ESP; darüber `MENU_PAGE_*`. Heutige PiDrive-Bäume (~100–200 Knoten) passen; große lokale Bibliotheken nicht ohne Lazy-Loading (R23 / A18).

**`uid_counter`:** nur erhöhen, wenn sich die UID-Menge ändert — nicht bei jedem Rebuild/`menu_rev` (sonst Permanent-Reload im Auto). Contract-Spiegel zu PiDrive M1.

Offen: Aktionen ohne Wiedergabe nach `PlayItem` (A18).

---

## Änderungshistorie

| Version | Datum | Kurz |
|---------|-------|------|
| V1.0 | 2026-09 | Erster Entwurf: Architektur, PDAP-Skizze, Hub-Pflicht, Coexistence-Gate |
| V1.1 | 2026-09 | Review-Nachzüge: Discovery, Rate, Migration, RAM, Betriebsmodi |
| V1.2 | 2026-09-15 | F1–F5: Stack offen (A17), Phase −1, Menü-Transport §2.24, Weg F |
| V2.0 | 2026-09-15 | Hub-Contract am Code (H-F1–H-F9), Partitionen/Flash-Budget, OTA-NVS-Defer, zweistufiger Update-Pfad; Teil A/B-Marken |

---

## Nächster Schritt nach Freigabe

1. Flash-Budget Upstream-Messung ([FLASH-BUDGET.md](FLASH-BUDGET.md)) + Phase −1 → **A17** entscheiden  
2. Dann: Zustandsübergänge + PDAP-Header + FreeRTOS-Prioritäten + Komponentenstruktur  
3. Phase-0-Firmware (Analyzer + A2DP-Testton + Hub-Register-Stub) — **nicht vorher**
