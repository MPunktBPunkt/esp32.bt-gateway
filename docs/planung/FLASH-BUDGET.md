# Flash-Budget — Messmethode & erste Zahlen

**Stand:** 2026-09-15  
**Anlass:** [AUFTRAG-CURSOR-3.md](AUFTRAG-CURSOR-3.md) §2.1/§2.2 · Risiko **R25** · Input für **A17**  
**Regel:** Keine Firmware in diesem Repo vor A17. Messung nur mit **Upstream-Beispielen** in einem Scratch-Verzeichnis außerhalb des Repos.

---

## 1. Harte Grenze

| Größe | Wert | Herkunft |
|-------|------|----------|
| Flash-Modul (Referenz) | 4 MB | Hub-Standard / D1 Mini (Q6) |
| App-Slot `ota_0` / `ota_1` | `0x1E0000` = **1 966 080 B** | Partitionstabelle §2.12a / Abschnitt D |
| Hub-Flash-Balken-Nenner | 1 966 080 | `iobroker.esp-hub` UI (`freeSketch / 1966080`) |
| Eskalationsschwelle | **&lt; 20 % Reserve** | Auftrag §2.2 → Owner, nicht still weiterbauen |

**App-Image muss unter 1 966 080 B bleiben** (inkl. Host-Stack, WiFi, LWIP, HTTP-Server, eingebettete WebUI, SBC, PDAP, ggf. Menü-Target). Dual-OTA auf 4 MB lässt keinen Spielraum für größere Slots ohne Factory-Opfer oder Modulwechsel (A16/Q6).

---

## 2. Partitionstabelle (Spezifikation, keine Datei)

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

- `0x3E0000 + 0x20000 = 0x400000` → 4 MB exakt.
- App-Offsets 64-KB-ausgerichtet.
- Kein `factory`-Slot; Recovery = USB-Merged @ `0x0`.
- `logs` (128 KB) nicht in App-Slots umwidmen — Reserve für Budget-Druck.

`partitions.csv` entsteht erst mit dem Phase-0-Skeleton nach A17.

---

## 3. Messmethode (verbindlich)

Scratch **außerhalb** dieses Repos, z. B. `/home/…/scratch/bt-gateway-flash-budget/`.

| Kandidat | Messobjekt |
|----------|------------|
| Bluedroid | IDF-Beispiel `bluetooth/bluedroid/classic_bt/a2dp_source` — (a) unverändert, (b) + WiFi STA + `esp_http_server` Minimal-Handler |
| BTstack | ESP32-Port, `a2dp_source_demo` bzw. AVRCP-Target-Demo — gleiche WiFi-Ergänzung |

Auswertung:

```bash
idf.py size
idf.py size-components
```

Erfassen:

1. Gesamtgröße der App (Flash)
2. Anteil BT-Stack / Controller
3. Anteil WiFi + LWIP + HTTP
4. **Reserve** = `(0x1E0000 − App) / 0x1E0000 × 100 %`

Kein Beispiel-Code ins Gateway-Repo kopieren — nur Zahlen und Methode hier.

**Unter 20 % Reserve → eskalieren** (R25), nicht Slot vergrößern und Dual-OTA opfern.

---

## 4. Erste Zahlen (Stand dieser Messung)

### 4.1 Upstream-IDF-Beispiele

| Build | Gesamt | BT | WiFi/LWIP/HTTP | Reserve vs. `0x1E0000` | Status |
|-------|--------|----|----------------|------------------------|--------|
| Bluedroid `a2dp_source` (stock) | — | — | — | — | **ausstehend** (kein ESP-IDF auf diesem Host) |
| Bluedroid + WiFi + `esp_http_server` | — | — | — | — | **ausstehend** |
| BTstack `a2dp_source_demo` (stock) | — | — | — | — | **ausstehend** |
| BTstack + WiFi + HTTP | — | — | — | — | **ausstehend** |

**Messstatus:** Methode und Budgetgrenze sind fest. Stack-Vergleichszahlen fehlen, weil auf dem Planungs-Host kein `idf.py` / ESP-IDF installiert ist. Die Messung ist **Pflicht-Input für A17** und darf nicht durch Spekulation ersetzt werden.

### 4.2 Familien-Proxy (nur Größenordnung, kein Stack-Vergleich)

Arduino-Familie, kein Classic-BT-Source — zeigt, was **WiFi + HTTP + WebUI** allein schon kosten:

| Artefakt | Bytes | % von `0x1E0000` | Hinweis |
|----------|------:|-----------------:|---------|
| `esp-hub-base.1.7.0.esp32.bin` | 1 201 152 | **61 %** | Hub-Referenz-App (Arduino), OTA-fähig |
| Reserve nach diesem Proxy | ≈ 764 928 | **39 %** | Rest für Classic-BT + SBC + PDAP + IDF-Overhead |

**Lesart:** Selbst ohne A2DP liegt die Familien-App bereits bei ~60 % des OTA-Slots. BTstack + voller Gateway-Umfang kann die Reserve unter 20 % drücken — deshalb Spalte „Codegröße“ in A17 und frühe Upstream-Messung.

### 4.3 WebUI-Nebenbedingung

Assets **einbetten** (`EMBED_FILES` / `EMBED_TXTFILES`), klein halten: kein Asset-Dateisystem, keine Fonts, keine Chart-Libs als Datei. Diagramme ggf. Inline-SVG wie `esp-hub-base` (§2.12).

---

## 5. Ergebnisort & Freigabe

| Frage | Ort |
|-------|-----|
| Messprotokoll / Tabellen-Update | diese Datei |
| Stack-Entscheidung inkl. Codegröße | [OFFENE-PUNKTE.md](OFFENE-PUNKTE.md) **A17** |
| Eskalation Modulgröße | **A16**, **Q6**, **R25** |
| Freigabe-Haken | Abschnitt **E**: „Flash-Budget gemessen (R25)“ |

---

## 6. Nächster Schritt

1. ESP-IDF (Zielversion nach A1) in Scratch installieren.  
2. Vier Builds aus §3 fahren, Tabelle §4.1 füllen.  
3. Bei Reserve &lt; 20 % → Owner (Q6: 8/16-MB-Modul?).  
4. Erst dann A17 finalisieren.
