# Pflichtenheft V1.0 — PiDrive Bluetooth Gateway (ESP32)

**Dokumentstatus:** Entwurf V1.0  
**Ziel:** Vollständige, implementierbare Spezifikation für ein eigenständiges ESP-IDF-Projekt + minimale PiDrive-Erweiterung.

---

## 2.1 Systemarchitektur & Verantwortungsgrenzen

### 2.1.1 Komponenten

| Komponente | Verantwortung | Kennt nicht |
|------------|---------------|-------------|
| **PiDrive** | Quellen, Audio-Engine, Trigger-Dispatcher, Metadaten-Erzeugung, Gateway-Client, Routing-Logik | A2DP, SBC, AVRCP, BlueZ (Gateway-Pfad) |
| **ESP32 Gateway** | WiFi, PDAP, Jitter-Buffer, SBC-Encoder, A2DP Source, AVRCP Target, Connection-Management, Analyzer, WebUI/Diagnose | DAB, Spotify, Menüs, Senderlisten, PiDrive-Logik |
| **BMW** | A2DP Sink + AVRCP Controller | Herkunft des Audios |

### 2.1.2 Datenflüsse

**Audio (Pi → ESP → BMW):**
```
Quelle → PipeWire → PCM (44,1 kHz / S16LE / Stereo)
→ PDAP Audio-Kanal → ESP Jitter-Buffer → SBC → A2DP → BMW
```

**Steuerung (BMW → ESP → Pi):**
```
BMW AVRCP → ESP Analyzer → PDAP Event → PiDrive Trigger-Dispatcher
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

- A2DP Sink + Source gleichzeitig (kein Handy-Relay)
- Mehrere gleichzeitige A2DP-Sinks
- AAC, aptX, LDAC
- HFP / Telefonie
- Audio-Mixing oder Quellen-Umschaltung im ESP
- BMW-spezifische Geschäftslogik im ESP
- ESP32-S3 oder andere BLE-only-Chips
- Vollständige BlueZ-Emulation
- Automatisches Resampling oder Clock-Drift-Korrektur (nur Überwachung)

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

**Gate-Test (vor jeder weiteren Arbeit):**

- ESP mit aktivem WiFi
- A2DP Source zum BMW
- ≥ 30 Minuten Dauerbetrieb
- Keine hörbaren Dropouts
- Kein unerwarteter Disconnect
- Messwerte: Packet-Loss, Jitter, Buffer-Underruns, RSSI, CPU-Last

**Bei Nichtbestehen:** Architekturänderung (z. B. externes WiFi-Modul oder anderer Ansatz) vor Fortsetzung.

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
- Sample-Rate: fest **44,1 kHz**
- Format: **16 Bit, Stereo, S16LE**
- Bitpool: parametrisierbar (Startwert 32–35), empirisch optimieren
- Nur **eine** gleichzeitige A2DP-Sink-Verbindung (BMW)

**SBC-Encoding** läuft auf dem ESP. Der Pi liefert ausschließlich PCM.

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

**Security (V1 minimal):** Pre-Shared-Key / Token im HELLO/AUTH.

---

## 2.9 AVRCP

- ESP = **AVRCP Target**
- BMW = **AVRCP Controller**
- V1: Vollständiger Analyzer (alles loggen)
- Später: Mapping der beobachteten Commands (NEXT, PREVIOUS, PLAY, PAUSE, STOP, VOL± …) auf PDAP-Events
- Keine Geschäftslogik im ESP – der Pi entscheidet die Bedeutung

---

## 2.10 Metadata-Engine

- Pi sendet strukturierte Metadata (Title, Artist, Album, Source, Station, Playing-Status …)
- ESP setzt daraus AVRCP TrackChanged + Element Attributes
- Zuerst observe (welche Attribute der BMW anfordert), dann implementieren

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
- PDAP Control
- PDAP Audio Receiver + Jitter-Buffer
- SBC Encoder
- A2DP Source
- AVRCP
- WebUI / HTTP
- Diagnostics / Watchdog
- State Machine

---

## 2.17 PiDrive-Integration

- Neuer `gateway_client`
- `audio route gateway` als zusätzliche Route
- Parallelbetrieb: `audio backend = bluez | gateway`
- `pidrive_avrcp.service` kann später umgestellt werden, zunächst parallel lassen
- Trigger-Dispatcher bleibt unverändert

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
| Integration | DAB/Webradio über Gateway hör- und steuerbar |
| Robustheit | Mehrere erfolgreiche Auto-Tests ohne manuellen Eingriff |

---

## 2.21 Hardware

- **Nur klassischer ESP32** (WROOM-32 / entsprechendes DevKit)
- Antenne für 2,4 GHz
- Stabile 5-V-Versorgung (Auto-tauglich)
- Kein ESP32-S3

---

## Nächster Schritt nach Freigabe

Ausarbeitung der detaillierten Zustandsübergänge + PDAP-Header-Definition + FreeRTOS-Task-Prioritäten, parallel Start der Phase-0-Firmware (Analyzer + A2DP-Testton).
