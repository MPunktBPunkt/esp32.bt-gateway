# Offene Punkte & Planungsentscheidungen

Stand: 2026-09-14 (Review V1.1). Ziel: vor Firmware-Start klären, was die Architektur wirklich festnagelt.

Nachzüge: [REVIEW-V1.1.md](REVIEW-V1.1.md).

---

## Empfohlene Reihenfolge (kurz)

```
1. Stack festlegen (ESP-IDF Bluedroid) + Hub-Client-Pfad     ← A1/A9
2. Phase 0 Skeleton: BT Classic + Logs + Hub register stub
3. Coexistence-Gate (WiFi + A2DP ≥ 30 min, inkl. Hub-Heartbeat)
4. AVRCP-Analyzer am realen BMW
5. PDAP-Header + Control-Kanal spezifizieren
6. PCM über UDP + Jitter-Buffer (+ Laptop-Tester)
7. Metadata observe → implement
8. PiDrive: PW-Sink + gateway_client + audio_output=gateway
9. DAB→PipeWire (eigenes PiDrive-Paket, falls Gateway-DAB nötig)
10. Hub-OTA hardened (deferred while STREAMING) + Bin in Hub-Ablage
```

Phase 0 und Coexistence **vor** PDAP-Vollausbau. PiDrive-Code erst nach stabilem Laptop→ESP→BMW-PCM.

**PiDrive-Ist-Analyse:** [PIDRIVE-INTEGRATION.md](PIDRIVE-INTEGRATION.md) (`pidrive` v0.11.127).  
**Hub-Integration:** [HUB-INTEGRATION.md](HUB-INTEGRATION.md).  
**Betriebsmodi / Handy / WiFi:** [BETRIEBSMODI.md](BETRIEBSMODI.md).

---

## A. Entscheidungen (bitte festlegen)

### A1. Firmware-Stack: ESP-IDF vs. PlatformIO/Arduino

| Option | Pro | Contra |
|--------|-----|--------|
| **ESP-IDF + Bluedroid** (Pflichtenheft) | Offizieller A2DP-Source-Pfad, volle BT-Classic-Kontrolle, Analyzer machbar | Weicht von `esp32.ergo` / Hub-Projekten ab; steilere Lernkurve; Hub-Client selbst bauen |
| PlatformIO + Arduino-BT-Libs | Vertrautes Setup, fertiger `HubClient` | A2DP **Source** + AVRCP Target + WiFi-Coexistence auf Classic-ESP ist dort dünn / fragil |

**Empfehlung:** ESP-IDF (wie im Pflichtenheft) **plus** dünner IDF-Hub-Client (USB-Flash + OTA über `iobroker.esp-hub`, kein Hub-Compile). Siehe [HUB-INTEGRATION.md](HUB-INTEGRATION.md). Die anderen `esp32.*`-Projekte bleiben Arduino/PlatformIO; dieses Repo ist bewusst „anders“, bleibt aber hub-programmierbar.

**Status:** □ offen / □ bestätigt

### A2. Coexistence-Plan B (falls Gate fällt)

Ideen priorisieren, **bevor** wir am Gate scheitern:

1. ESP32 Classic nur BT; WiFi über separates Modul (SPI/UART, z. B. ESP32-C3 als WiFi-Client) — teurer, klarer
2. Ethernet statt WiFi am Gateway (W5500 o. Ä.) — im Auto ggf. unpraktisch
3. Reduktion: WebUI nur im Setup-Modus, PDAP über schmalen Kanal — riskant, Gate-Kriterium bleibt
4. Anderer Chip mit besserer Coexistence — bricht „nur klassischer ESP32“

**Empfehlung:** Plan B1 skizzieren, aber nicht bauen, bis Gate gemessen ist.

**Status:** □ offen / □ bestätigt

### A3. Control-Transport: TCP vs. WebSocket

- TCP: einfacher, robust, gut für eingebettete Clients
- WebSocket: WebUI und Pi können denselben Kanal nutzen; etwas mehr Overhead

**Empfehlung V1:** TCP für PDAP Control; WebUI separat HTTP(+WS nur für Live-Logs). Audio bleibt UDP.

**Status:** □ offen / □ bestätigt

### A4. Auth / Pairing-Modell BMW

- Feste MAC / Bonding-Datei auf ESP?
- Wer initiiert Connect — ESP immer, oder nur auf Pi-Command?
- Sichtbarer BT-Name: `PiDrive` / `esp32.bt-gateway` / konfigurierbar?

**Empfehlung:** Konfigurierbarer Name; Bonding persistent in NVS; Connect-Policy: auto-reconnect zu gebondetem Gerät, manuell über WebUI/PDAP überschreibbar.

**Status:** □ offen / □ bestätigt

### A5. Repo-Scope PiDrive

PiDrive-Client (`gateway_client`) lebt:

- a) nur Doku/API-Contract hier, Code in `pidrive`, oder
- b) `clients/pidrive/` als Stub in diesem Repo?

**Empfehlung (nach Code-Review):** **a)** — Contract + `clients/pdap_tester/` (Python/Laptop) hier; produktiver Client als `pidrive/integration/gateway_client.py` neben `avrcp_trigger.py`. Ankerpunkte `audio.py` / `map_event` / MPRIS liegen nur in `pidrive`.

**Status:** □ offen / □ bestätigt

### A6. PCM-Contract: wer resample’t?

PiDrive-Quellen sind format-gemischt (FM 32 kHz mono, Webradio oft 44.1/48, DAB oft außerhalb PW).

| Option | Pro | Contra |
|--------|-----|--------|
| **Pi erzwingt 44.1 Stereo S16LE** (PW-Converter) | ESP bleibt dumm; passt Pflichtenheft | Capture-Node + DAB-Bridge nötig |
| ESP akzeptiert mehrere Formate | flexibler | mehr CPU/Komplexität auf ESP |

**Empfehlung:** Pi erzwingt Format (siehe PIDRIVE-INTEGRATION §4.4).

**Status:** □ offen / □ bestätigt

### A7. DAB im Gateway-Pfad?

Heute: `dab_play.py` → Direct ALSA / Klinke, **kein** zuverlässiger BT-Transport.

- a) Gateway-V1 ohne DAB (Webradio/Spotify/lokal zuerst)
- b) DAB parallel auf PW umbauen (PiDrive-Änderung, eigenes Risiko)

**Empfehlung:** V1 = **a**; DAB als Folgepaket.

**Status:** □ offen / □ bestätigt

### A8. Zwei AVRCP-Targets zum BMW vermeiden

Bei `audio_output=gateway` darf Pi-BlueZ **nicht** gleichzeitig AVRCP/A2DP zum selben BMW halten (sonst Doppel-Events: `avrcp_trigger` + MPRIS + ESP).

**Empfehlung:** Gateway aktiv → BMW-Auto-Connect auf Pi idle; `pidrive_avrcp.service` nur für BlueZ-Fallback-Pfad relevant.

**Status:** □ offen / □ bestätigt

### A9. ESP-Hub-Programmierung (Anforderung bestätigt)

Der ESP soll über `iobroker.esp-hub` programmierbar sein (USB-Flash + OTA-Push + Geräte-Status).

| Option | Bewertung |
|--------|-----------|
| Arduino-`esp-hub-base` als Basis | kollidiert mit Classic-BT-Source |
| **IDF-`HubClient` + Hub-USB/OTA** | passt; Compile-Tab entfällt |
| Hub nur optional / später | widerspricht Anforderung |

**Empfehlung:** Pflicht ab Feldgerät; Skeleton-Modul schon in Phase 0 mitplanen. Details: [HUB-INTEGRATION.md](HUB-INTEGRATION.md).

**Status:** □ offen / ☑ Anforderung gesetzt (Umsetzungsweg: IDF-Client)

### A10. WiFi-Betriebsmodus (STA / SoftAP / APSTA)

ESP32 kann **APSTA** (Client + SoftAP). Mit Classic-BT eng.

| Situation | Empfehlung |
|-----------|------------|
| Carport / Heim | STA → Heim-WLAN (Hub + Pi) |
| Auto mit Pi | STA → **Pi-Hotspot** (bevorzugtes Auto-Muster) |
| Auto ohne Pi-Netz | SoftAP, Clients joinen ESP |
| APSTA dauerhaft | nur nach eigenem Gate-Test |

**Empfehlung:** `wifi_mode=auto` (bekanntes STA-Netz → STA, sonst SoftAP); APSTA opt-in. Siehe [BETRIEBSMODI.md](BETRIEBSMODI.md).

**Status:** □ offen / □ bestätigt

### A11. Zweiter PDAP-Client (Handy direkt)

| Option | Bewertung |
|--------|-----------|
| V1: nur ein Audio-Client, zweiter → busy | einfach, stabil |
| Priority PiDrive > Gast | etwas mehr Logik |
| Mixing mehrerer Clients | **Nein** (Nicht-Ziel) |

**Empfehlung:** V1 busy-reject; Handy-Musik primär über PiDrive. PDAP-Gast (Laptop/Phone) als einzelner Ersatz-Client ok.

**Status:** □ offen / □ bestätigt

### A12. AVRCP-Ziel wenn Gast-PDAP aktiv

Wenn kein PiDrive, aber Handy/Laptop PDAP-Session: Events an Session-Owner. Wenn PiDrive online: Events immer an PiDrive (`map_event`).

**Empfehlung:** PiDrive Prefer; sonst Session-Owner.

**Status:** □ offen / □ bestätigt

### A13. Discovery Pi↔ESP

| Option | V1 |
|--------|-----|
| `gateway_host` Config auf Pi | Pflicht |
| mDNS `bt-gateway.local` | empfohlen |
| SoftAP-SSID-Muster | Setup/Auto |
| Nur Hub-UI ablesen | Diagnose |

**Empfehlung:** Config + mDNS; SoftAP für Setup. Siehe REVIEW L1.

**Status:** □ offen / □ bestätigt

### A14. A2DP 44,1 vs. 48 kHz

**Empfehlung:** Negotiation auf 44,1 erzwingen; sonst ESP-Resample 44,1→48. Siehe REVIEW L2.

**Status:** □ offen / □ bestätigt

### A15. BT-Anzeigename am BMW

`PiDrive` (vertraut) vs. `PiDrive-GW` / `esp32.bt-gateway` (klarer Migration).

**Empfehlung:** konfigurierbar, Default `PiDrive-GW` während Parallelbetrieb, später optional `PiDrive`.

**Status:** □ offen / □ bestätigt

### A16. Hardware-Variante bei RAM-Engpass

WROOM zuerst; bei Heap/Underruns WROVER/PSRAM (kein S3).

**Status:** □ offen / □ bestätigt

---

## B. Technische Risiken (bewusst tracken)

| # | Risiko | Früher Nachweis |
|---|--------|-----------------|
| R1 | WiFi+BT Classic Dropout | Gate-Test 30 min mit Testton |
| R2 | BMW akzeptiert ESP als A2DP Source nicht / odd Codec-Negotiation | Phase 0 Connect-Log |
| R3 | AVRCP-Subset ungewöhnlich (nur Pressed, kein Released, Volume-Pfad) | Analyzer-Logs über Zündzyklen |
| R4 | SBC CPU-Last + WiFi → Underruns | KPI: CPU, buffer underruns |
| R5 | Clock-Drift Pi↔ESP hörbar nach langer Fahrt | Drift-Anzeige V1, Korrektur später |
| R6 | Auto-Strom / Reset-Verhalten | Versorgung + Brownout-Tests |
| R7 | Pi-Quellen ohne festes PCM → Underruns/Pitch | Capture-Node erzwingen; Tester mit 32 kHz |
| R8 | DAB Direct-ALSA → kein Gateway-Audio | Explizites DAB-Paket oder V1 ohne DAB |
| R9 | Dual AVRCP (BlueZ+ESP) → Doppeltrigger | Policy: ein Target zum BMW |
| R10 | Absolute Volume / DSP-Konflikte | Wie PiDrive: Vol± als Events, kein Absolutkampf |
| R11 | Hub-OTA während Stream | `ota_allowed` nur außerhalb STREAMING |
| R12 | Merged- vs. App-Bin im Hub verwechselt | Release-Checkliste + Naming |
| R13 | SoftAP/APSTA + A2DP Dropout | Gate-Tests §2.4 erweitern; Fallback Pi-Hotspot |
| R14 | Zwei PDAP-Clients kämpfen um Session | busy-reject / PiDrive-Priority |
| R15 | A2DP 48 kHz vs. PCM 44,1 → Pitch/Reject | Force 44,1 oder ESP-Resample |
| R16 | RAM/CPU WROOM zu eng (BT+WiFi+SBC+Buffer) | KPIs; Plan B WROVER |
| R17 | Altes Pi-Bonding am BMW + Dual-Connect | Migrations-Checkliste §2.22 |

---

## C. Was bewusst noch *nicht* spezifiziert wird

Bis Phase-0-Messung liegen:

- Konkrete AVRCP→Trigger-Mapping-Tabelle
- Finale Bitpool- / Buffer-Defaults (nur Startwerte)
- Exakte FreeRTOS-Prioritäten und Core-Pinning
- Byte-genaue PDAP-Structs (nur Header-Skizze vorhanden)

Diese Dateien kommen als Nächstes nach Freigabe:

- `docs/planung/ZUSTANDSAUTOMAT.md`
- `docs/planung/PDAP.md`
- `docs/planung/FREERTOS.md`

---

## D. Vorgeschlagene Ordnerstruktur (Firmware, später)

```
esp32.bt-gateway/
├── docs/planung/          ← jetzt
├── main/                  ← ESP-IDF app
├── components/
│   ├── hub_client/        ← POST /api/register + otaUrl (Familie-Contract)
│   ├── pdap/
│   ├── a2dp_source/
│   ├── avrcp_target/
│   ├── jitter_buffer/
│   ├── analyzer/
│   └── webui/
├── clients/
│   └── pdap_tester/       ← Laptop/Python Referenzclient
└── tools/
```

Noch nicht anlegen, bis Stack (A1) bestätigt ist — `hub_client` aber mit Phase‑0-Skeleton mitplanen.

---

## E. Freigabe-Checkliste Pflichtenheft

- [ ] Konzept inhaltlich OK
- [ ] Nicht-Ziele OK
- [ ] Phase 0 + Coexistence als erste Arbeitspakete OK
- [ ] A1–A16 entschieden oder bewusst vertagt
- [ ] PiDrive-Integrationsplan OK ([PIDRIVE-INTEGRATION.md](PIDRIVE-INTEGRATION.md))
- [ ] Hub-Programmierung OK ([HUB-INTEGRATION.md](HUB-INTEGRATION.md))
- [ ] Betriebsmodi / Handy / WiFi OK ([BETRIEBSMODI.md](BETRIEBSMODI.md))
- [ ] Review-Nachzüge OK ([REVIEW-V1.1.md](REVIEW-V1.1.md))
- [ ] Danach: Zustandsautomat + PDAP-Header + Phase-0-Skeleton
