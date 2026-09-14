# ESP-Hub-Integration — Programmieren & OTA

**Stand:** 2026-09-14  
**Anforderung:** Der Gateway-ESP muss über [iobroker.esp-hub](https://github.com/MPunktBPunkt/iobroker.esp-hub) (Port **8093**) sichtbar sein und programmierbar/updatebar sein — analog zur übrigen `esp32.*`-Familie.  
**Stack-Entscheidung bleibt:** ESP-IDF + Bluedroid (Classic BT). Hub-Anbindung = **dünner HTTP-Client**, nicht Arduino-`esp-hub-base`-Kopie.

Vertragsreferenz Adapter: `iobroker.esp-hub/Schnittstellen.md`.

---

## 1. Was „mit dem Hub programmieren“ hier heißt

| Pfad | Nutzung für bt-gateway | Pflicht? |
|------|------------------------|----------|
| **USB-Flash** aus Hub-UI (`esptool`, typ. ab `0x0`) | Erstflash / Recovery; vorgefertigte IDF-`.bin` in Hub-Firmware-Ablage | **ja** |
| **OTA-Push** aus Hub-UI (`otaUrl` in Heartbeat-Response) | Feld-Updates ohne USB | **ja** |
| **Geräte-Liste / Status** (`POST /api/register`) | MAC, Version, RSSI, KPIs, optional `ios` | **ja** |
| **Compile-Tab** (arduino-cli) | Arduino-only — für IDF **nicht** vorgesehen | **nein** |
| **Lokales Web-OTA** am ESP (`POST /ota-upload` o. Ä.) | Recovery, wenn Hub unerreichbar | empfohlen |

Kein Serial-Tunnel über einen anderen ESP — der Hub-Host flasht USB-direkt bzw. pusht OTA per WLAN.

---

## 2. Architekturentscheidung

| Option | Bewertung |
|--------|-----------|
| A) Volles `esp-hub-base` / Arduino-Familie | **Nein** — kollidiert mit A2DP Source / AVRCP Target (Bluedroid) |
| B) Arduino-Companion + IDF-App | **Nein** — zu komplex, Coexistence schlechter |
| **C) Dünner IDF-`HubClient`** | **Ja** — Register + OTA + optionale `ios`; PDAP/WebUI/BT bleiben nativ IDF |
| D) Hub erst später | Widerspricht der Anforderung; Phase‑0 darf lokal flashen, aber Hub-Modul von Anfang an mitplanen |

**Ergebnis:** ESP-IDF-Firmware implementiert denselben HTTP-Contract wie `HubClient` (z. B. `esp32.ergo` / `esp32.heartrate`), inkl. **OTA erst wenn erlaubt** (während `STREAMING` deferren — Muster `setOtaAllowed` in ergo).

```
ioBroker esp-hub :8093
        │  USB esptool  /  GET /firmware/*.bin  /  otaUrl
        │  POST /api/register  (≈30 s)
        ▼
ESP32 bt-gateway (ESP-IDF)
  ├── HubClient (HTTP)          ← Familie
  ├── WebUI + lokales OTA       ← Familie-ähnlich
  ├── PDAP (PiDrive)            ← gateway-spezifisch
  └── A2DP Source + AVRCP       ← gateway-spezifisch
```

Zwei Heartbeats, klar getrennt:

- **Hub-Heartbeat** (~30 s): Inventar / OTA-Trigger  
- **PDAP-Heartbeat** (1–2 s): Pi↔Gateway Audio-Session  

---

## 3. Firmware-Mindestumfang (IDF)

1. **Partitionstabelle** mit OTA-Slots (Factory + OTA_0/OTA_1 oder dual OTA) — App-Image muss in den OTA-Slot passen  
2. **NVS:** `hub_host`, `hub_port` (Default 8093), Gerätename, Token falls später  
3. **`POST /api/register`** mind. Felder:
   - Pflicht/Identität: `mac`, `name`, `hwType: "esp32"`, `version`, `ip`, `rssi`, `uptime`, `freeHeap`
   - UI-wichtig: `chipModel`, `freeSketch`
   - Familie: `fwType: "bt-gateway"`, optional `board`
   - Optional `ios`: Gateway-Status (BMW connected, A2DP streaming, buffer %, WiFi RSSI, PDAP online, …)
4. Response: `interval` beachten; bei `otaUrl` → OTA starten **nur wenn** `ota_allowed()` (nicht mitten im Stream / Analyzer-kritisch optional konfigurierbar)
5. **OTA:** ESP-IDF `esp_http_client` + `esp_ota` / `esp_https_ota` — App-Image vom Hub (`GET /firmware/...`)
6. **Bin-Naming** (Hub-Familie-Check): `bt-gateway.<semver>.esp32.bin`  
   - USB-Flash: oft **merged** Image ab `0x0`  
   - Hub-OTA: **App-only** Image für OTA-Partition — beide Artefakte in CI/Release erzeugen und dokumentieren
7. Adapter-seitig: Bin in `iobroker.esp-hub/firmware/` ablegen bzw. über UI hochladen — **kein** Adapter-Code nötig für Basis-Flash/OTA

---

## 4. Was bewusst anders bleibt als bei `esp-hub-base`

| Thema | Hub-Familie (Arduino) | bt-gateway |
|-------|----------------------|------------|
| Build | arduino-cli / PlatformIO Arduino | **ESP-IDF** |
| Hub-Compile-Tab | ja | **nein** |
| BT | oft NimBLE / kein Classic Source | **Bluedroid Classic Source** |
| Heartbeat-Payload | Sensor-`ios` | Gateway-/BMW-/Buffer-`ios` |
| OTA unter Last | ergo deferred | **pflicht: deferred while STREAMING** |

---

## 5. Arbeitspakete

| # | Paket | Wann |
|---|-------|------|
| H1 | Partitionstabelle OTA-fähig + Build-Artefakte (merged + app) | mit Phase‑0-Skeleton |
| H2 | IDF-`components/hub_client/` (register + otaUrl) | früh, parallel Analyzer |
| H3 | Hub-Host in NVS / WebUI-Setup | mit WebUI |
| H4 | `ota_allowed()` an State Machine koppeln | vor Feld-OTA |
| H5 | Bin namenskonform in Hub-Ablage / Release-Doku | vor erstem Feldgerät |
| H6 | Optional: lokales Web-OTA | Recovery |

Phase‑0 (Labor): `idf.py flash` bleibt ok. **Vor Auto-Dauerbetrieb:** H1–H4 fertig, damit Updates über den Hub gehen.

---

## 6. Risiken

| Risiko | Mitigation |
|--------|------------|
| OTA während A2DP → Dropout / Brick-Gefühl | OTA nur außerhalb `STREAMING`; WebUI-Warnung |
| Merged-Bin vs. App-Bin verwechselt | Release-Checkliste + Hub-Dateiname/Docs |
| Coexistence: Hub-Traffic | Heartbeat klein (~30 s); Gate-Test inkl. Hub-Register |
| falsche Chip-Familie | nur `.esp32.bin`, nie S3 (passt zur Hub-Safety) |

---

## 7. Referenzen (Implementierer)

- `iobroker.esp-hub/Schnittstellen.md` — HTTP-Contract  
- `iobroker.esp-hub/README.md` — USB / OTA / Naming  
- `ESP32.esp-hub/esp-hub-base.ino` — Referenzverhalten  
- `esp32.ergo/src/core/HubClient.cpp` — deferred OTA  
- `nodes/esp32.heartrate/src/core/HubClient.*` — modularer Client  
- Dieses Repo: Pflichtenheft § Hub, `OFFENE-PUNKTE` A1/A9
