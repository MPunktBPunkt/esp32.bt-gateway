# Offene Punkte & Planungsentscheidungen

Stand: 2026-09-14. Ziel: vor Firmware-Start klären, was die Architektur wirklich festnagelt.

---

## Empfohlene Reihenfolge (kurz)

```
1. Stack festlegen (ESP-IDF Bluedroid)     ← Entscheidungsbedarf
2. Phase 0 Skeleton: BT Classic + Logs    ← Labor / Kopfhörer zuerst
3. Coexistence-Gate (WiFi + A2DP ≥ 30 min) ← hartes Go/No-Go
4. AVRCP-Analyzer am realen BMW
5. PDAP-Header + Control-Kanal spezifizieren
6. PCM über UDP + Jitter-Buffer
7. Metadata observe → implement
8. PiDrive gateway_client + Parallelbetrieb
```

Phase 0 und Coexistence **vor** PDAP-Vollausbau. Sonst riskieren wir Wochen Protokollarbeit auf einer Hardware-Kombi, die im Auto nicht hält.

---

## A. Entscheidungen (bitte festlegen)

### A1. Firmware-Stack: ESP-IDF vs. PlatformIO/Arduino

| Option | Pro | Contra |
|--------|-----|--------|
| **ESP-IDF + Bluedroid** (Pflichtenheft) | Offizieller A2DP-Source-Pfad, volle BT-Classic-Kontrolle, Analyzer machbar | Weicht von `esp32.ergo` / Hub-Projekten ab; steilere Lernkurve |
| PlatformIO + Arduino-BT-Libs | Vertrautes Setup | A2DP **Source** + AVRCP Target + WiFi-Coexistence auf Classic-ESP ist dort dünn / fragil |

**Empfehlung:** ESP-IDF (wie im Pflichtenheft). Die anderen `esp32.*`-Projekte bleiben Arduino/PlatformIO; dieses Repo ist bewusst „anders“, weil Classic-BT Source das verlangt.

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

**Empfehlung:** Contract + Referenz-Tesclient (Python/Laptop) hier; produktiver `gateway_client` in `pidrive`.

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

Noch nicht anlegen, bis Stack (A1) bestätigt ist.

---

## E. Freigabe-Checkliste Pflichtenheft

- [ ] Konzept inhaltlich OK
- [ ] Nicht-Ziele OK
- [ ] Phase 0 + Coexistence als erste Arbeitspakete OK
- [ ] A1–A5 entschieden oder bewusst vertagt
- [ ] Danach: Zustandsautomat + PDAP-Header + Phase-0-Skeleton
