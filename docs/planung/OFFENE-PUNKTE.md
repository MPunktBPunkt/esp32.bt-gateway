# Offene Punkte & Planungsentscheidungen

Stand: 2026-09-15 (Planung **V2.0+** — Auftrag 4: Pfad-Mapping, Flash-Proxy, P-F1–P-F5). Ziel: vor Firmware-Start klären, was die Architektur wirklich festnagelt.

Nachzüge: [REVIEW-V1.1.md](REVIEW-V1.1.md), [REVIEW-V1.2.md](REVIEW-V1.2.md), [REVIEW-V1.3.md](REVIEW-V1.3.md), [REVIEW-V1.4.md](REVIEW-V1.4.md).  
Aufträge: [AUFTRAG-CURSOR-2.md](AUFTRAG-CURSOR-2.md), [AUFTRAG-CURSOR-3.md](AUFTRAG-CURSOR-3.md), [AUFTRAG-CURSOR-4.md](AUFTRAG-CURSOR-4.md).  
Messplan: [PHASE-0-MESSPLAN.md](PHASE-0-MESSPLAN.md) · Flash: [FLASH-BUDGET.md](FLASH-BUDGET.md).

---

## Owner-Fragen (Q12–Q13 neu)

| # | Frage | Blockiert |
|---|-------|-----------|
| Q1–Q5 | siehe Auftrag 2 | A17/A18 |
| Q6–Q11 | siehe Auftrag 3 | A16, OTA-Policy, A21 |
| **Q12** | Wo werden ESP32-Projekte gebaut? (Planungs-Host ohne Toolchain) | Flash-Messung → **A17** → Firmware |
| **Q13** | Menükanal von Anfang an paginiert spezifizieren, oder erst nach Phase −1? | §2.24 / A18 |

Fahrzeug-Pi als Build-Host **nicht** empfohlen (Auftrag 4).

---

## Empfohlene Reihenfolge (kurz)

```
0. Phase −1 am Pi (Browsing/Metadata-Probe)                  ← blockiert A17/A18
0b. Flash-Budget: Upstream-Beispiele messen (R25)            ← Input für A17
1. A1 (ESP-IDF) bestätigt halten; A17 Host-Stack entscheiden + A9 Hub (bestätigt)
2. Erst dann Komponentenstruktur / Phase-0-Skeleton
3. Coexistence-Gate (WiFi + A2DP ≥ 30 min, inkl. Hub-Heartbeat; Stack-spezifisch)
4. AVRCP-Analyzer am realen BMW (Phase 0)
5. PDAP-Header + Control-Kanal (+ optional Menü-Kanal bei S3)
6. PCM über UDP + Jitter-Buffer (+ Laptop-Tester)
7. Metadata observe → implement (stackabhängig, F1)
8. PiDrive: PW-Sink + gateway_client + audio_output=gateway
9. DAB→PipeWire (eigenes PiDrive-Paket, falls Gateway-DAB nötig)
10. Hub-OTA hardened (NVS-Defer while STREAMING) + Stufe-2-Depot (A21)
```

Phase −1 **und** Flash-Messung **vor** A17. Phase 0 und Coexistence **vor** PDAP-Vollausbau. Keine Firmware-Ordner, solange A17 offen ist. PiDrive-Code erst nach stabilem Laptop→ESP→BMW-PCM.

**PiDrive-Ist-Analyse:** [PIDRIVE-INTEGRATION.md](PIDRIVE-INTEGRATION.md) (`pidrive` v0.11.127).  
**Hub-Integration:** [HUB-INTEGRATION.md](HUB-INTEGRATION.md).  
**Betriebsmodi / Handy / WiFi:** [BETRIEBSMODI.md](BETRIEBSMODI.md).  
**AVRCP / Menü:** [AVRCP-MOEGLICHKEITEN.md](AVRCP-MOEGLICHKEITEN.md).

---

## A. Entscheidungen (bitte festlegen)

### A1. Firmware-Build: ESP-IDF vs. PlatformIO/Arduino

| Option | Pro | Contra |
|--------|-----|--------|
| **ESP-IDF** (Build + FreeRTOS) | Offizieller Classic-BT-Pfad, Analyzer machbar, Hub-Client selbst baubar | Weicht von `esp32.ergo` / Hub-Projekten ab; steilere Lernkurve |
| PlatformIO + Arduino-BT-Libs | Vertrautes Setup, fertiger `HubClient` | A2DP **Source** + AVRCP Target + WiFi-Coexistence auf Classic-ESP ist dort dünn / fragil |

**Empfehlung:** ESP-IDF **plus** dünner IDF-Hub-Client (USB-Flash + OTA über `iobroker.esp-hub`, kein Hub-Compile). Siehe [HUB-INTEGRATION.md](HUB-INTEGRATION.md).

**V1.2-Korrektur:** Die frühere Empfehlung „ESP-IDF + **Bluedroid**“ ist mit F1/F2 (keine Target-Metadaten-API, kein Browsing) **nicht mehr haltbar**, solange Display/Menü Ziel sind. A1 bleibt die **Build-/RTOS-Entscheidung**. Der **Bluetooth-Host-Stack** ist herausgelöst → **A17**.

**Status:** □ offen / □ bestätigt (Tendenz: ESP-IDF)

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

### A4. Auth / Pairing-Modell BMW — inkl. Bestätigungsdialog

- Feste MAC / Bonding-Datei auf ESP?
- Wer initiiert Connect — ESP immer, oder nur auf Pi-Command?
- Sichtbarer BT-Name: `PiDrive` / `esp32.bt-gateway` / konfigurierbar?
- **Praxis (Owner):** Beim Koppeln Handy↔iDrive wartet das Fahrzeug auf eine **aktive Bestätigung am Handy**. Ohne die Dialog-Bestätigung kommt keine BT-Verbindung zustande. Der ESP hat kein Display — derselbe Dialog muss woanders landen.

**Empfehlung — zweistufig:**

| Phase | Verhalten |
|-------|-----------|
| **Erst-Pairing** | Expliziter, zeitlich begrenzter **Pairing-Modus** (WebUI / SoftAP-Setup / PDAP). ESP discoverable. SSP-Anfrage vom BMW → **Code + Gerätename in der WebUI** (und `ios.pairing`). Nutzer bestätigt in der WebUI (= Ersatz für den Handy-Dialog) und am iDrive, falls das Auto ebenfalls fragt. Timeout ~60–120 s → Pairing abbrechen, nicht still auto-akzeptieren außerhalb des Modus. |
| **Danach (Alltag)** | Bonding-Keys in **NVS**. Reconnect ohne erneute Bestätigung. Neue Pairing-Anfragen **ablehnen**, solange Pairing-Modus aus (Schutz vor Fremdgeräten während der Fahrt). |

**IO-Capabilities:** Zuerst messen (Phase 0), welches SSP-Verfahren das NBT tatsächlich verlangt (Numeric Comparison / Passkey / Just Works). WebUI-Confirm deckt Numeric Comparison und Passkey ab. Just Works nur **innerhalb** des Pairing-Fensters auto-accept — nie dauerhaft „alles annehmen“.

**Connect-Policy:** Auto-Reconnect zum gebondeten BMW; manuell über WebUI/PDAP überschreibbar. Name konfigurierbar (A15), Default während Migration `PiDrive-GW`.

**Status:** □ offen / □ bestätigt (Richtung: WebUI als Handy-Dialog-Ersatz)

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

### A9. ESP-Hub-Programmierung — **bestätigt**

Der ESP soll über `iobroker.esp-hub` programmierbar sein (USB-Flash + OTA-Push + Geräte-Status).

| Option | Bewertung |
|--------|-----------|
| Arduino-`esp-hub-base` als Basis | kollidiert mit Classic-BT-Source |
| **IDF-`HubClient` + Hub-USB/OTA** | passt; Compile-Tab entfällt |
| Hub nur optional / später | widerspricht Anforderung |

**Verifiziert** gegen `iobroker.esp-hub` **v0.5.12** (`main.js`): OTA = Pull aus einmaliger `otaUrl` in Heartbeat-Antwort; USB-Flash frei wählbare Adresse Default `0x0` (Merged-Image ohne Adapteränderung); `chipModel` für Familien-Sperre; `ios` als JSON-String. Details: [AUFTRAG-CURSOR-3.md](AUFTRAG-CURSOR-3.md) Kap. 1 (H-F1–H-F9), [HUB-INTEGRATION.md](HUB-INTEGRATION.md), [REVIEW-V1.3.md](REVIEW-V1.3.md).

**Status:** ☑ **bestätigt** (Umsetzungsweg: IDF-Client, keine Adapteränderung)

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

### A16. Hardware-Variante bei RAM-/Flash-Engpass

WROOM zuerst; bei Heap/Underruns WROVER/PSRAM (kein S3). Bei S3 + Menübaum + BTstack + WiFi wird der Heap enger. Flash: 4 MB Dual-OTA ist Spezifikation; 8/16-MB-Modul nur nach R25-Eskalation (Q6) — weicht von Hub-Referenzhardware ab.

**Status:** □ offen / □ bestätigt

### A17. Bluetooth-Host-Stack (herausgelöst aus A1)

Entscheidungsvorlage — **erst nach Phase −1 und Flash-Budget-Messung** festlegen. Siehe [AVRCP-MOEGLICHKEITEN.md](AVRCP-MOEGLICHKEITEN.md) §4, [PHASE-0-MESSPLAN.md](PHASE-0-MESSPLAN.md), [FLASH-BUDGET.md](FLASH-BUDGET.md).

| Option | Metadaten | Browsing | Aufwand | Risiko | Codegröße |
|--------|-----------|----------|---------|--------|-----------|
| Bluedroid unverändert | ⛔ | ⛔ | gering | Display bleibt leer → Projektziel verfehlt (F1/F2) | Messung ausstehend — typisch kleiner als BTstack |
| Bluedroid + Patch in `bta_av_act.c` | ⚠ per Patch | ⛔ | mittel | Fork-Pflege bei jedem IDF-Update; S3 dauerhaft verbaut | ähnlich Bluedroid + Patch-Overhead |
| **BTstack auf ESP32** | ✅ API | ✅ API | höher (neuer Stack, eigener Coexistence-Nachweis) | Controller-Eigenheiten ESP32-Port; Lizenz nicht-kommerziell (R21/R22) | **kritisch:** kann mit WiFi+WebUI OTA-Budget reißen (R25) |

**Regel:** Eine Stack-Entscheidung **ohne** Größenmessung ist unzulässig. Unter 20 % Reserve vs. `0x1E0000` → Owner eskalieren (Q6), nicht still Dual-OTA opfern.

**Empfehlung:** **BTstack**, sobald Phase −1 zeigt, dass Metadaten im Fahrzeug ankommen — und **zwingend**, falls Browsing möglich ist — **sofern** das Flash-Budget trägt. Lizenzfrage Q2 vorher klären.

**Status:** □ offen (blockiert durch Phase −1 **und** Flash-Messung) / □ bestätigt

### A18. Menü-Transport und Semantik (bei S3)

Wenn S3 kommt: Abbildung Ordner / Sender / Aktionen / Schalter auf Folder-Items und Media-Elements; Aktionen, nach denen das Auto Wiedergabe erwartet (`PlayItem`); Baum vollständig vs. seitenweise (`GetFolderItems` ist paginiert); `uid_counter`-Invalidierung ohne Permanent-Reload bei jedem BT-Statuswechsel.

**Vorschlag:** Seitenbasiertes Abrufmodell von Anfang an (P-F5 / golden tree ≈ 98 KB; Q13). `uid_counter` nur bei Änderung der UID-Menge — gespiegelt aus PiDrive-Auftrag M1. Pi: paginierter Export (**P13**).

**Status:** □ offen (nach Phase −1 / S3-Entscheidung / Q13) / □ bestätigt

### A19. Multi-Source über den Pi (nicht ESP-Relay)

Formal: Handy/Tablet koppeln am **Pi als A2DP-Sink** (F4 — Sink+Source gleichzeitig auf ESP unmöglich). PipeWire führt in denselben Capture-Punkt wie DAB/Webradio; Umschaltung über bestehendes PiDrive-Menü. Folge: PiDrive wird AVRCP **Controller** gegenüber dem Handy und muss Skip vom BMW durchreichen → neuer Fall in `map_event()` / Event-Contract ([PIDRIVE-INTEGRATION.md](PIDRIVE-INTEGRATION.md) §2.9/§8; Arbeitspaket in `pidrive` G3).

**Aufwand (P-F3):** Installation setzt WirePlumber auf `bluez5.roles = [ a2dp_source ]` only (`install.sh` inline; `pipewire-config/` tot). Sink-Rolle = **kein Nullaufwand** — WP-Rollen erweitern (**P15**), nicht „bereits erledigt“.

**Status:** □ offen / □ bestätigt (Richtung mit Eigentümer abgestimmt laut Auftrag)

### A20. AVRCP-Anzeigename und Player-Identität

Bei S3 erscheint der Gateway in der Player-Liste des Autos. Zusammen mit A15 (BT-Anzeigename): Wie heißt das Gerät im iDrive? Während Parallelbetrieb BlueZ+ESP — zwei Player sichtbar?

**Empfehlung:** mit A15 bündeln; Default-Name während Migration `PiDrive-GW`.

**Status:** □ offen / □ bestätigt

### A21. Feld-Update-Pfad Stufe 2 (PiDrive als Firmware-Depot)

Im Fahrzeug ist der Hub nicht erreichbar (R24). Hub-OTA (Stufe 1) funktioniert nur im Carport/Heim-WLAN.

| Option | Bewertung |
|--------|-----------|
| **(a) Stufe 2:** Pi hält Image vor, `POST /ota-upload` am Gateway (PSK, gleiche Prüfungen wie Hub-OTA) | empfohlen — kabellos auch unterwegs |
| (b) nur Carport-Updates; im Auto USB | widerspricht „kabellos“ zur Hälfte |
| (c) APSTA mit Route nach Hause | fragil, Coexistence, oft keine Route |

**Empfehlung:** **(a)**. Gateway-Contract in [PIDRIVE-INTEGRATION.md](PIDRIVE-INTEGRATION.md); Pi-Implementierung (`pidrivectl gateway firmware fetch|list|push`) im **`pidrive`-Repo** nachziehen — hier nur spezifizieren.

**Status:** □ offen / □ bestätigt (Empfehlung a; Owner-Frage Q11)

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
| R11 | Hub-OTA während Stream / URL verloren | NVS-Persistenz + `OTA_PENDING`; nie URL verwerfen |
| R12 | Merged- vs. App-Bin im Hub verwechselt | Naming + App-Desc-Magic `0xABCD5432` @ `0x20` |
| R13 | SoftAP/APSTA + A2DP Dropout | Gate-Tests §2.4 erweitern; Fallback Pi-Hotspot |
| R14 | Zwei PDAP-Clients kämpfen um Session | busy-reject / PiDrive-Priority |
| R15 | A2DP 48 kHz vs. PCM 44,1 → Pitch/Reject | Force 44,1 oder ESP-Resample |
| R16 | RAM/CPU WROOM zu eng (BT+WiFi+SBC+Buffer) | KPIs; Plan B WROVER |
| R17 | Altes Pi-Bonding am BMW + Dual-Connect | Migrations-Checkliste §2.22 |
| R18 | Bluedroid-TG liefert keine Metadaten → BMW-Display leer | F1 verifiziert; Entscheidung A17 |
| R19 | NBT Evo öffnet keinen Browsing-Kanal → S3 entfällt | Phase −1 |
| R20 | Sink+Source auf einem ESP32 unmöglich → Multi-Source nur über Pi | F4 verifiziert; A19 |
| R21 | BTstack-ESP32-Port: Controller-Eigenheiten, Coexistence unbekannt | Gate §2.4 mit BTstack wiederholen |
| R22 | BTstack-Lizenz (nicht-kommerziell) kollidiert mit späterer Weitergabe | vor A17 klären; LICENSE/README |
| R23 | Menübaum sprengt WROOM-RAM (große lokale Listen) | Größenbudget A18; Lazy-Loading |
| R24 | Hub im Fahrzeug nicht erreichbar; OTA-URL enthält Heim-IP | Stufe 2 + lokales Web-OTA Pflicht (A21) |
| R25 | App-Image überschreitet OTA-Slot `0x1E0000` | Frühe Messung ([FLASH-BUDGET.md](FLASH-BUDGET.md)); eingebettete WebUI; Eskalation &lt; 20 % → A16/Q6; **Slots nicht vergrößern**; Proxy Arduino pessimistisch — kein A16 auf Proxy-Basis |
| R26 | Pi-Metadaten nur bei Menü-`rev` (P-F1) → Display tot im Gateway-Pfad | **P12** vor Metadata-Abnahme |
| R27 | Vollbaum ≈ 98 KB ohne Paging (P-F5) → Heap / PDAP | Paginiertes Menü Pflicht; Nicht-Ziel Volltransfer |
| R28 | iDrive wartet auf Pairing-Bestätigung am Remote (Handy-Dialog) — ESP ohne Display | Zeitlich begrenzter Pairing-Modus + WebUI-Confirm (A4); Bonding NVS; Phase 0 SSP-Verfahren messen |

---

## C. Was bewusst noch *nicht* spezifiziert wird

Bis Phase −1 / Phase-0-Messung / A17 / Flash-Messung:

- Konkrete AVRCP→Trigger-Mapping-Tabelle
- Finale Bitpool- / Buffer-Defaults (nur Startwerte)
- Exakte FreeRTOS-Prioritäten und Core-Pinning
- Byte-genaue PDAP-Structs (nur Header-Skizze; Menü-Kanal erst recht nur Skizze)
- Komponentenliste unter `components/` (hängt an A17)
- Gemessene Codegrößen Bluedroid vs. BTstack (Methode steht)

Diese Dateien kommen als Nächstes nach Freigabe / A17:

- `docs/planung/ZUSTANDSAUTOMAT.md`
- `docs/planung/PDAP.md`
- `docs/planung/FREERTOS.md`
- `docs/RELEASE.md` (Checkliste vor erstem Feldgerät)

---

## D. Partitionstabelle & Ordnerstruktur (Firmware, später)

### D.1 Partitionstabelle — **festgeschrieben** (Spezifikation)

Keine `partitions.csv` vor A17. Offsets später ändern = USB-Flash im Feld.

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

Rechnung: `0x3E0000 + 0x20000 = 0x400000` (4 MB exakt). App-Slots 64-KB-ausgerichtet. Kein Factory. `logs` nicht in App umwidmen. Details: [FLASH-BUDGET.md](FLASH-BUDGET.md), Pflichtenheft §2.12a.

### D.2 Vorgeschlagene Ordnerstruktur

```
esp32.bt-gateway/
├── docs/planung/          ← jetzt
├── main/                  ← ESP-IDF app
├── components/
│   ├── hub_client/        ← POST /api/register + otaUrl (Familie-Contract)
│   ├── ota_manager/       ← NVS-Defer, App-Desc, Rollback, /ota-upload
│   ├── pdap/
│   ├── …                  ← A2DP/AVRCP: Bluedroid-Module ODER BTstack-Profile + menu_target
│   ├── jitter_buffer/
│   ├── analyzer/
│   └── webui/
├── clients/
│   └── pdap_tester/       ← Laptop/Python Referenzclient
└── tools/
```

**Noch nicht anlegen**, bis A17 entschieden ist. `hub_client` / `ota_manager` mit Phase‑0-Skeleton mitplanen, sobald Stack steht. Bei BTstack heißen die BT-Komponenten anders als `a2dp_source`/`avrcp_target`-Eigenbau.

---

## E. Freigabe-Checkliste Pflichtenheft

- [ ] Konzept inhaltlich OK
- [ ] Nicht-Ziele OK (inkl. Multi-Source über Pi, F4)
- [ ] Phase −1 + Phase 0 + Coexistence als erste Arbeitspakete OK
- [ ] A1, A17–A21 und A2–A16 entschieden oder bewusst vertagt
- [ ] **Teil A/B-Marken im Pflichtenheft gesetzt** (V2.0)
- [ ] **Hub-Contract gegen `main.js` verifiziert (A9)**
- [ ] **Flash-Budget gemessen (R25)** — Methode steht; Upstream-Zahlen noch ausstehend
- [ ] **Update-Pfad Stufe 2 entschieden (A21 / Q11)**
- [ ] PiDrive-Integrationsplan OK ([PIDRIVE-INTEGRATION.md](PIDRIVE-INTEGRATION.md))
- [ ] Hub-Programmierung OK ([HUB-INTEGRATION.md](HUB-INTEGRATION.md))
- [ ] Betriebsmodi / Handy / WiFi OK ([BETRIEBSMODI.md](BETRIEBSMODI.md))
- [ ] AVRCP-Möglichkeiten OK ([AVRCP-MOEGLICHKEITEN.md](AVRCP-MOEGLICHKEITEN.md))
- [ ] Review-Nachzüge OK ([REVIEW-V1.1.md](REVIEW-V1.1.md), [REVIEW-V1.2.md](REVIEW-V1.2.md), [REVIEW-V1.3.md](REVIEW-V1.3.md))
- [ ] Danach: Zustandsautomat + PDAP-Header + Phase-0-Skeleton (nach A17)
