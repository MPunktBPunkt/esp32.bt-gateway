# Pflichtenheft V1.0 — PiDrive Bluetooth Gateway (ESP32)

**Dokumentstatus:** Entwurf V1.1 (Planungs-Review)  
**Ziel:** Vollständige, implementierbare Spezifikation für ein eigenständiges ESP-IDF-Projekt + minimale PiDrive-Erweiterung.  
**Review-Nachzüge:** [REVIEW-V1.1.md](REVIEW-V1.1.md)

---

## 2.1 Systemarchitektur & Verantwortungsgrenzen

### 2.1.1 Komponenten

| Komponente | Verantwortung | Kennt nicht |
|------------|---------------|-------------|
| **PiDrive** | Quellen, Audio-Engine, Trigger-Dispatcher, Metadaten-Erzeugung, Gateway-Client, Routing-Logik | A2DP, SBC, AVRCP, BlueZ (Gateway-Pfad) |
| **ESP32 Gateway** | WiFi, PDAP, Jitter-Buffer, SBC-Encoder, A2DP Source, AVRCP Target, Connection-Management, Analyzer, WebUI/Diagnose, **ESP-Hub Heartbeat/OTA** | DAB, Spotify, Menüs, Senderlisten, PiDrive-Logik |
| **iobroker.esp-hub** | Geräteinventar, USB-Flash, Firmware-Ablage, OTA-Push | Audio, BT, PDAP, BMW |
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

- A2DP Sink + Source gleichzeitig (kein Handy-**BT**-Relay)
- Mehrere gleichzeitige A2DP-Sinks
- Mehrere gleichzeitige PDAP-Audio-Sessions (ein Session-Owner)
- AAC, aptX, LDAC
- HFP / Telefonie
- Audio-Mixing oder Quellen-Umschaltung im ESP
- BMW-spezifische Geschäftslogik im ESP
- ESP32-S3 oder andere BLE-only-Chips
- Vollständige BlueZ-Emulation
- Automatisches Resampling oder Clock-Drift-Korrektur (nur Überwachung)
- Dauerhaftes APSTA+A2DP ohne bestandenes Gate (siehe Betriebsmodi)

**Erlaubt / geplant (nicht Nicht-Ziel):** Handy-Musik **über PiDrive** (Spotify Connect etc.) oder später als **PDAP-Client** (WLAN/SoftAP). Details: [BETRIEBSMODI.md](BETRIEBSMODI.md).

---

## 2.3 Phase 0 – BMW Bluetooth Profil-Discovery & Analyzer (höchste Priorität)

**Ziel:** Das konkrete Verhalten des NBT Evo vermessen, bevor die Audio-Pipeline fertiggestellt wird.

**Anforderungen an den ESP:**

- Discovery, Pairing und Connect zum BMW
- Vollständiges Logging aller eingehenden AVRCP-PDUs (Pass-Through, Register Notification, Get Element Attributes, Volume, etc.)
- Logging von Timing (Pressed/Released), Reihenfolge und Parametern
- A2DP-Verbindungsaufbau und Codec-Negotiation loggen
- RSSI, Disconnect-Reasons, Connection-Interval
- WebUI + exportierbare Logs (JSON/CSV)
- Self-Test-Modus (Testton vom ESP direkt zum BMW)

**Exit-Kriterium Phase 0:**  
Vollständige Protokollierung des realen BMW-Verhaltens über mehrere Zündungszyklen und Fahrsituationen. Daraus abgeleitete Mapping-Tabelle für AVRCP und Metadata.

---

## 2.4 WiFi + Classic-BT Coexistence (Gate-Kriterium)

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

**Backoff-Strategie bei Disconnect:** 1 s → 2 s → 5 s → 10 s → 30 s → 60 s

**Regel:** „BMW disconnected“ (Zündung aus) ist ein erwarteter Zustand und löst keinen Fehleralarm aus.

---

## 2.6 A2DP-Audioarchitektur

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

- Buffer-Größe: parametrisierbar (20 / 40 / 60 / 80 / 100 / 150 / 200 ms)
- WebUI zeigt aktuellen Füllstand, Target, Underruns, Overruns, Packet-Loss, Jitter
- Bei Packet-Loss: Silence-Frame oder letzte Frame wiederholen (kein Knacken)
- **Audio-Clock-Drift:** In V1 nur überwachen und anzeigen. Keine Korrektur (Resampling/Frame-Drop). Architektur muss spätere Korrektur erlauben.

---

## 2.8 PDAP – PiDrive Gateway Protocol

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

**HELLO / Capabilities (V1):** neben AUTH/VERSION meldet der ESP u. a. `fw_version`, `bt_name`, `capabilities` (z. B. `audio_pcm_44100_s16le`, `metadata_v1`, `events_v1`), `max_buffer_ms`. Client bricht bei Inkompatibilität klar ab.

**Security (V1 minimal):** Pre-Shared-Key / Token im HELLO/AUTH.

**Discovery (Pi findet ESP):** Config `gateway_host` auf dem Pi (Pflicht-Minimum); empfohlen mDNS `bt-gateway.local` im STA-LAN; SoftAP-SSID-Muster für Setup/Auto. Details: [REVIEW-V1.1.md](REVIEW-V1.1.md) L1.

---

## 2.9 AVRCP

- ESP = **AVRCP Target**
- BMW = **AVRCP Controller**
- V1: Vollständiger Analyzer (alles loggen)
- Später: Mapping der beobachteten Commands auf PDAP-Events mit **PiDrive-Event-Namen**:
  `next`, `previous`, `play`, `pause`, `play_pause`, `stop`, `volumeup`, `volumedown`, `fast_forward`, `rewind`
- Keine Geschäftslogik im ESP – der Pi mappt via bestehendem `map_event()` (Menü vs. FM vs. DAB)
- Double-Tap-Semantik (1,2 s → `cat:0`) bleibt auf dem Pi
- Absolute Volume: nicht gegen BMW kämpfen (PiDrive-Praxis: Volume fix / DSP im Auto)
---

## 2.10 Metadata-Engine

- Pi sendet strukturierte Metadata analog zu den heutigen MPRIS-Feldern (Title, Artist, Album, Source, Playing-Status …)
- Felder stammen aus denselben Inputs wie `mpris2.update()` / Playback-Status / DLS / ICY
- ESP setzt daraus AVRCP TrackChanged + Element Attributes
- Zuerst observe (welche Attribute der BMW anfordert), dann implementieren
- Bei aktivem Gateway ist BlueZ-MPRIS fürs BMW-Display optional idle
---

## 2.11 Control API & Statusmodell

Der Pi behandelt den ESP wie ein Netzwerkgerät.

Mindest-Status:

- WiFi (RSSI, verbunden)
- Bluetooth (State, RSSI, Codec, Bitpool, MTU, Reconnect-Count, Disconnect-Reason)
- Audio (Streaming, Buffer-Level, Underruns/Overruns)
- Source / Track (von Pi gespiegelt)

---

## 2.12 WebUI

Funktionen:

- Live-Status (BMW, A2DP, AVRCP, Buffer, WiFi)
- AVRCP-Analyzer-Anzeige
- Self-Test (Testton)
- Buffer- und SBC-Parameter
- Logs / Export
- Verbindung manuell steuern (Connect / Disconnect / Reconnect)
- Hub-Host/Port konfigurieren; optional lokales OTA-Upload (Recovery)

---

## 2.12a ESP-Hub (Programmieren & OTA) — Pflicht

Vollständig: [HUB-INTEGRATION.md](HUB-INTEGRATION.md).

**Muss:**

- Periodischer `POST /api/register` an `iobroker.esp-hub` (Contract wie Familie)
- `fwType: "bt-gateway"`, `hwType: "esp32"`, SemVer, `chipModel`, `freeSketch`
- Optional `ios` mit Gateway-KPIs (BMW/A2DP/Buffer/PDAP)
- Auswertung von `otaUrl` → HTTP-OTA in OTA-Partition (**nur wenn nicht STREAMING** / `ota_allowed`)
- Release-Bins: `bt-gateway.<semver>.esp32.bin` — USB-Flash aus Hub-UI (merged) + OTA (app image)
- Dual-OTA-fähige Partitionstabelle

**Muss nicht:**

- Hub-Compile-Tab (arduino-cli) — Build bleibt ESP-IDF außerhalb des Adapters
- Arduino-`esp-hub-base` als Firmware-Grundlage

**Darf nicht:**

- OTA mitten in aktiver A2DP-Session ohne explizite Override-Policy

---

## 2.13 Diagnose, Logging & KPIs

Mindest-KPIs (WebUI + API + Log):

- WiFi: RSSI, Packet-Loss, Jitter
- Buffer: Level, Underruns, Overruns
- SBC: Frames, Drop-Rate, CPU
- BT: States, Codec, Bitpool, MTU, RSSI, Disconnect-Reasons
- High-Level: Source, Track, Playing

---

## 2.14 Recovery & Fehlerkonzept

- WiFi-Verlust → Gateway meldet `GATEWAY_AUDIO_LOST` → PiDrive entscheidet über Fallback (AUX/HDMI/Stop)
- BMW-Disconnect → Backoff + Reconnect-Versuch
- Buffer-Underrun → Silence + Status
- ESP-Reset → möglichst schneller Wiederaufbau der Bluetooth-Seite

---

## 2.15 Security (V1)

- Pre-Shared-Key / Token zwischen Pi und ESP
- Keine offenen Commands ohne Authentifizierung

---

## 2.16 ESP-IDF / FreeRTOS-Architektur (V1-Skizze)

Empfohlene Tasks (Prioritäten später festlegen):

- WiFi / Network
- Hub Client (register / OTA-gate)
- PDAP Control
- PDAP Audio Receiver + Jitter-Buffer
- SBC Encoder
- A2DP Source
- AVRCP
- WebUI / HTTP (+ lokales OTA)
- Diagnostics / Watchdog
- State Machine

---

## 2.17 PiDrive-Integration

Detailliert: [PIDRIVE-INTEGRATION.md](PIDRIVE-INTEGRATION.md) (Analyse `pidrive` v0.11.127).

Mindestumfang im Repo **`pidrive`** (nicht hier):

- Neuer `integration/gateway_client.py` (+ optional `systemd/pidrive_gateway.service`)
- `audio_output = gateway` als zusätzliche Route in `modules/audio.py` / `settings.py`
- PipeWire: virtueller Sink + Resample auf Contract-PCM
- Parallelbetrieb: `audio_output = bt | gateway` (BlueZ-BMW-Pfad idle, wenn Gateway aktiv)
- Reverse: PDAP-Events → `map_event()` → `/tmp/pidrive_cmd` (Trigger-Dispatcher unverändert)
- Metadata-Push parallel zu / statt MPRIS-BMW-Pfad
- **DAB:** separates Arbeitspaket — heute Direct-ALSA; ohne PW-Bridge kein Gateway-DAB

In **diesem** Repo: PDAP-Contract + `clients/pdap_tester/` (Laptop-Referenz).
---

## 2.18 Testplan (Labor)

1. ESP A2DP Source → Kopfhörer (Testton)
2. ESP + WiFi → BMW (Coexistence-Gate)
3. AVRCP-Analyzer mit realem BMW
4. Metadata-Test
5. Netzwerk-Audio (Laptop → ESP → BMW)
6. Vollständige PDAP-Strecke

---

## 2.19 Auto-Testplan

- Zündungszyklen
- Lange Fahrten (30–60 min)
- Störquellen (andere WLAN/BT-Geräte)
- Schnelles NEXT/PREV
- Quellenwechsel
- WiFi-Ausfall-Simulation
- ESP-Reset während Betrieb

---

## 2.20 Exit-Kriterien je Phase

| Phase | Exit-Kriterium |
|-------|----------------|
| 0 | Vollständige BMW-Protokollvermessung + Analyzer |
| Coexistence | ≥ 30 min stabil A2DP + WiFi |
| A2DP | Stabiler Testton zum BMW |
| AVRCP | Alle relevanten Commands geloggt und gemappt |
| Metadata | Titel/Artist erscheinen korrekt |
| PDAP Audio | Stabiles PCM über WLAN |
| Integration | Webradio (und ggf. Spotify/lokal) über Gateway hör- und steuerbar; DAB optional nach PW-Bridge |
| Robustheit | Mehrere erfolgreiche Auto-Tests ohne manuellen Eingriff |

---

## 2.21 Hardware

- **Nur klassischer ESP32** (Ziel: WROOM-32 / DevKit; bei RAM-Engpass Plan B: **WROVER/PSRAM**, weiterhin kein S3)
- Antenne für 2,4 GHz (Platzierung im Auto beachten)
- Stabile 5-V-Versorgung (Auto-tauglich; Verpolung/Lastabwurf hardwareseitig bedenken)
- Brownout-Detector + Task-Watchdog aktiv
- Optional später: Status-LED, Factory-Reset-Taster (V1: Reset über WebUI reicht)
- Kein ESP32-S3

---

## 2.22 Ersteinrichtung & Migration (BMW)

1. ESP flashen (Hub-USB oder `idf.py`), WiFi/Hub/PSK konfigurieren  
2. Am BMW altes Pi-/Dongle-Gerät entfernen oder nicht parallel verbinden  
3. ESP pairen (Name konfigurierbar, Bonding in NVS)  
4. Pi: `gateway_host` + `audio_output=gateway`; BlueZ-Auto-Connect zum BMW aus  
5. Coexistence-Gate + Webradio-Test vor DAB-Umbau  

---

## 2.23 Latenz, Logs, Reset (Kurz)

- **Latenz:** Ziel Ende-zu-Ende grob &lt; 150–200 ms (Buffer parametrisierbar); Anzeige über Timestamps  
- **Logs:** RAM-Ring + WebUI-Export; Persistenz optional LittleFS  
- **Factory-Reset:** WebUI löscht NVS (WiFi, Bonding, Tokens); SoftAP mit WPA2-Passwort  

---

## Nächster Schritt nach Freigabe

Ausarbeitung der detaillierten Zustandsübergänge + PDAP-Header-Definition + FreeRTOS-Task-Prioritäten, parallel Start der Phase-0-Firmware (Analyzer + A2DP-Testton + Hub-Register-Stub).
