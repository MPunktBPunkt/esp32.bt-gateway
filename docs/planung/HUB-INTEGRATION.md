# ESP-Hub-Integration — Programmieren & OTA

**Stand:** 2026-09-15 (V2.0 / Auftrag Cursor-3)  
**Anforderung:** Der Gateway-ESP muss über [iobroker.esp-hub](https://github.com/MPunktBPunkt/iobroker.esp-hub) (Port **8093**) sichtbar und programmierbar/updatebar sein — analog zur übrigen `esp32.*`-Familie.  
**Stack-Entscheidung:** ESP-IDF + Host-Stack **A17** (offen). Hub-Anbindung = **dünner HTTP-Client**, nicht Arduino-`esp-hub-base`-Kopie.

**Wahrheitsquelle:** `iobroker.esp-hub/main.js` (**v0.5.12** lokal verifiziert).  
`Schnittstellen.md` (v0.1.0) ist unvollständig und **nicht** die Contract-Quelle ([AUFTRAG-CURSOR-3.md](AUFTRAG-CURSOR-3.md) Kap. 1).

---

## 1. Was „mit dem Hub programmieren“ hier heißt

| Pfad | Nutzung für bt-gateway | Pflicht? |
|------|------------------------|----------|
| **USB-Flash** aus Hub-UI (`esptool write_flash <addr> <file>`, Default **`0x0`**) | Erstflash / Recovery; **Merged**-IDF-Image | **ja** (H-F6) |
| **OTA-Push** aus Hub-UI → einmalige `otaUrl` in Heartbeat-Antwort → ESP **pullt** per HTTP | Feld-Updates im Heim-WLAN | **ja** (H-F1/H-F2) — Stufe 1 |
| **Geräte-Liste / Status** (`POST /api/register`) | MAC, Version, RSSI, KPIs, `ios` | **ja** |
| **Compile-Tab** (arduino-cli) | Arduino-only — für IDF **nicht** vorgesehen | **nein** |
| **Lokales `POST /ota-upload`** am ESP | Feld-Updates **ohne Heim-WLAN** (PiDrive-Depot) | **ja** — Stufe 2 (A21/R24) |

Kein Serial-Tunnel über einen anderen ESP — der Hub-Host flasht USB-direkt bzw. liefert die OTA-URL per Heartbeat.

**Topologie:** Hub-OTA funktioniert **nur im Carport/Heim-WLAN**. Im Auto (SoftAP oder Pi-Hotspot) gibt es keine Route zum Hub; die OTA-URL enthält zudem die Adapter-Heim-IP (`main.js` OTA-Push). Deshalb Stufe 2.

---

## 2. Verifizierte Hub-Fakten (H-F1–H-F9)

| # | Befund | `main.js` (v0.5.12) | Konsequenz |
|---|--------|---------------------|------------|
| **H-F1** | OTA ist **Pull**: `otaUrl` in Heartbeat-Antwort; ESP lädt per HTTP (kein TLS) | Register-Antwort ~297–309; Firmware-Serve `/firmware/` | IDF: `esp_https_ota` mit `CONFIG_OTA_ALLOW_HTTP=y` oder `esp_http_client` + `esp_ota_*`. Kein ArduinoOTA/espota/3232 |
| **H-F2** | `otaUrl` ist **einmalig** — State wird nach Einfügen in die Antwort gelöscht | ~306–309 | Sofort NVS-Persistenz; nie URL verwerfen bei Defer (STREAMING) |
| **H-F3** | `GET /api/ota/check?mac=` liest denselben State, **löscht nicht** | ~1166–1175 | Nachfrage, solange Heartbeat die URL noch nicht konsumiert hat |
| **H-F4*** | **Korrektur vs. Auftrag v0.5.8:** ab v0.5.12 wird nicht-leeres `name` bei **jedem** Heartbeat übernommen („Device is source of truth“) | ~252–256 | Erstname trotzdem korrekt setzen (A15/A20); Umbenennung über Gerät möglich |
| **H-F5** | Chip-Familien-Sperre: Dateiname vs. `chipModel`; unbekannte Seite → erlaubt | ~78–95, ~1236–1241 | `chipModel` ist Schutzfunktion, nicht Kosmetik |
| **H-F6** | USB-Flash: `write_flash <addr> <file>`, Default `0x0` | ~820–834 | Merged-Image Erstflash ohne Adapteränderung |
| **H-F7** | Flash-Balken: `freeSketch / 1966080` | ~2380 | OTA-Slot `0x1E0000` = korrekte Skala |
| **H-F8** | `ios` = ein JSON-String-State, UI als Liste | ~268–269, ~2369+ | KPIs ohne Adaptercode sichtbar |
| **H-F9** | `/api/register` ohne Auth; MAC 12 Hex | Sanitize + Register | Heimnetz; keine Security-Cmds über Hub |

\*Auftrag Cursor-3 zitierte v0.5.8 mit sticky first name; aktueller Code weicht ab — Im Zweifel gilt der Code.

---

## 3. Architekturentscheidung

| Option | Bewertung |
|--------|-----------|
| A) Volles `esp-hub-base` / Arduino-Familie | **Nein** — kollidiert mit A2DP Source / AVRCP Target |
| B) Arduino-Companion + IDF-App | **Nein** |
| **C) Dünner IDF-`HubClient` + `ota_manager`** | **Ja** — Register + NVS-OTA + `ios`; PDAP/WebUI/BT bleiben IDF |
| D) Hub erst später | Widerspricht Anforderung |

```
ioBroker esp-hub :8093          (nur Heim-WLAN)
        │  USB esptool @0x0  /  GET /firmware/*.bin  /  otaUrl einmalig
        │  POST /api/register  (≈30 s)
        ▼
ESP32 bt-gateway (ESP-IDF)
  ├── HubClient (HTTP)          ← Familie
  ├── ota_manager (NVS, Defer, App-Desc, Rollback)
  ├── WebUI + POST /ota-upload  ← Stufe 2 (PiDrive)
  ├── PDAP
  └── A2DP Source + AVRCP
```

Zwei Heartbeats, klar getrennt:

- **Hub-Heartbeat** (~30 s): Inventar / OTA-Trigger  
- **PDAP-Heartbeat** (1–2 s): Pi↔Gateway Audio-Session  

---

## 4. Firmware-Mindestumfang (IDF) — nach A17

1. **Partitionstabelle** Dual-OTA ohne Factory, Slots je `0x1E0000` — siehe [FLASH-BUDGET.md](FLASH-BUDGET.md) / OFFENE-PUNKTE D  
2. **NVS:** `hub_host`, `hub_port` (8093), PSK, BT-Name, Bonding, `pending_ota_*`  
3. **`POST /api/register`** Felder: `mac`, `name`, `hwType:"esp32"`, `chipModel`, `version`, `ip`, `rssi`, `uptime`, `freeHeap`, `freeSketch`, `fwType:"bt-gateway"`, `ios` inkl. `otaState`  
4. Bei `otaUrl` → **sofort NVS**, dann `ota_allowed()`; sonst `OTA_PENDING`  
5. **OTA-Prüfungen vor Write:** App-Desc Magic `0xABCD5432` @ Offset `0x20`; `Content-Length` ≤ Slot-Größe; Rollback aktiv  
6. **Bins:** `bt-gateway.<semver>.usb.esp32.bin` (Merged @ `0x0`) und `bt-gateway.<semver>.esp32.bin` (App-only). Nie `.esp32s3.bin`  
7. Adapter: Bin ablegen — **kein** Adapter-Code nötig (A9 bestätigt)

---

## 5. Arbeitspakete (ersetzt früheres §5)

| # | Paket | Wann |
|---|-------|------|
| **H1** | Partitionstabelle in Spezifikation (§2.12a / Abschnitt D) | **erledigt** (V2.0) |
| **H2** | [FLASH-BUDGET.md](FLASH-BUDGET.md) Methode + Zahlen | Methode jetzt; Upstream-Zahlen vor A17 |
| **H3** | A9 bestätigt; A17+Codegröße; A21/R24/R25 | **erledigt** in OFFENE-PUNKTE |
| **H4** | §2.12a V2.0; dieses Dokument an H-F* | **erledigt** |
| **H5** | `/ota-upload`-Contract in PIDRIVE-INTEGRATION | **erledigt** (Spezifikation) |
| **H6** | `components/hub_client/` | nach A17 |
| **H7** | `components/ota_manager/` (NVS, Defer, App-Desc, Rollback, Backoff) | nach A17, vor Feld-OTA |
| **H8** | SoftAP-Setup + NVS-Config + WLAN-Logs | nach A17, mit WebUI |
| **H9** | `ota_allowed()` + `POST /ota-upload` | nach A17, vor Feldgerät |
| **H10** | `docs/RELEASE.md` Checkliste | vor erstem Feldgerät |

Phase‑0 Labor: `idf.py flash` ok. Vor Auto-Dauerbetrieb: H6–H9.

---

## 6. Risiken

| Risiko | Mitigation |
|--------|------------|
| OTA während A2DP / URL verloren | NVS-Persistenz; OTA nur außerhalb `STREAMING`; `otaState` in `ios` |
| Merged-Bin vs. App-Bin | Naming + Magicword-Prüfung @ `0x20` (nicht nur `0xE9`) |
| Hub im Auto unerreichbar | Stufe 2 / lokales OTA (R24, A21) |
| App zu groß für Slot | Flash-Budget-Messung; Eskalation R25; Slots nicht vergrößern |
| Coexistence: Hub-Traffic | Heartbeat klein (~30 s); Gate inkl. Register |
| Falsche Chip-Familie | immer `chipModel` senden; nur `.esp32.bin` |

---

## 7. Referenzen (Implementierer)

- `iobroker.esp-hub/main.js` — **Contract** (Zeilen siehe Tabelle §2)  
- `iobroker.esp-hub/README.md` — USB / OTA / Naming (Ergänzung, nicht Wahrheit)  
- `Schnittstellen.md` — **nicht** als Quelle zitieren  
- `ESP32.esp-hub/esp-hub-base.ino` — Referenzverhalten Familie  
- `esp32.ergo` / `nodes/esp32.heartrate` — deferred OTA / HubClient (URL nur RAM → **nicht** 1:1 kopieren; Gateway braucht NVS)  
- Dieses Repo: Pflichtenheft §2.12a, OFFENE-PUNKTE A9/A21, [AUFTRAG-CURSOR-3.md](AUFTRAG-CURSOR-3.md)
