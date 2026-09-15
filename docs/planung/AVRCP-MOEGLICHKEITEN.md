# AVRCP-Möglichkeiten — BT-Steuerung des iDrive

**Dokumentstatus:** Planung V1.2 (Ideenfindung)  
**Anlass:** [AUFTRAG-CURSOR-2.md](AUFTRAG-CURSOR-2.md) · Befunde F1–F5  
**Fahrzeug-Wahrheit:** `pidrive/iDriveBt.md` §2/§3 und `pidrive/RUNTIME_FLOWS.md` Abschnitt I — dort referenzieren, nicht hier duplizieren.  
**Gegenstück PiDrive:** `pidrive/AUFTRAG-MENUE-UND-GATEWAY.md`

---

## 1. Rollenbild

| Rolle | Gerät | AVRCP | A2DP |
|-------|--------|-------|------|
| **Controller + Sink** | BMW (NBT Evo) | Controller (befiehlt, fordert Metadaten/Browse an) | Sink (spielt) |
| **Target + Source** | ESP Gateway (Ziel) | Target (antwortet, liefert Meta/Browse) | Source (sendet SBC) |
| **heute (BlueZ)** | Raspberry Pi | Target (via BlueZ) | Source (via BlueZ) |

Der Gateway ersetzt die BlueZ-Strecke zum BMW. PiDrive bleibt Gehirn: Quellen, Menüsemantik, Trigger. Der ESP spricht nur Bluetooth-Classic mit dem Auto und PDAP mit dem Client.

---

## 2. Drei Kanäle — was sie für die Bedienung liefern

### 2.1 AVCTP Control (PSM 0x0017)

Transport der klassischen Pass-Through-Kommandos und der Metadata-PDUs.

| Was ankommt / geht | Nutzen für Bedienung |
|--------------------|----------------------|
| Pass-Through (Play/Pause, Next/Prev, Vol±, …) | Lenkrad-/iDrive-Medien-Tasten → PDAP-Events → Pi `map_event()` |
| Register Notification / Track Changed | Display-Refresh anstoßen |
| GetElementAttributes / SetAbsoluteVolume | Titelzeilen; Lautstärke-Politik (nicht gegen BMW kämpfen) |

**Ist am NBT Evo (laut PiDrive):** zuverlässig vor allem Skip und Play/Pause. Die physische iDrive-Einheit (`MENU`, `BACK`, `OPTION`, Dreh-Drück) steuert die **BMW-eigene** Oberfläche und wird **nicht** als AVRCP weitergereicht. Sichtbar sind drei Textzeilen (Titel / Interpret / Album) — immer nur der **markierte** Eintrag, nie die Liste. Daraus entstand die heutige Skip=Cursor-/Play=Enter-Bedienung inkl. „Zurueck“-Eintrag und Bestätigungsebenen.

→ Ausbaustufe **S1**.

### 2.2 AVCTP Browsing (PSM 0x001B)

Echter Listen-Kanal: `SetBrowsedPlayer`, `ChangePath`, `GetFolderItems`, `GetItemAttributes`, `Search`, `PlayItem`, `GetTotalNumberOfItems`.

| Was er liefern würde | Nutzen |
|----------------------|--------|
| Ordner-/Item-Liste im iDrive gerendert | Dreh-Drück bedient die **BMW-Liste**, nicht Skip-als-Cursor |
| PlayItem auf UID | Explizite Aktivierung ohne Doppeldeutigkeit Skip/Play |

**Voraussetzung Target:** Stack mit Browsing-API (BTstack hat sie; Bluedroid nicht — F2).  
**Voraussetzung Fahrzeug:** BMW muss Browsing als Controller öffnen — **messbar in Phase −1**, ohne ESP-Firmware (F5: BlueZ bewirbt Browsing im SDP, lehnt aber die meisten Scopes ab → der Versuch des Autos ist sichtbar).

→ Ausbaustufe **S3** (Zielbild, wenn Messung positiv).

### 2.3 Cover-Art / BIP / OBEX (AVRCP 1.6+)

Albumcover über separaten OBEX-Pfad. PiDrive meldet seit v0.11.126 korrekt „erst ab 1.6“. Für V1/V1.2 **kein** Planungsziel; hier nur als Grenze notiert: auch bei S3 gibt es keine Freitexteingabe und kein Cover, solange BIP nicht spezifiziert und gemessen ist.

---

## 3. Ausbaustufen S1–S3

| Stufe | Transport | Was der Fahrer sieht | Voraussetzung |
|-------|-----------|----------------------|---------------|
| **S1** heute | AVRCP Metadata (3 Zeilen) | markierter Eintrag; Skip/Play | Target kann Metadaten setzen (F1 → Stack!) |
| **S2** | S1 + Player Application Settings | zusätzliche Auswahlwerte als BMW-Einstellungen | BMW-abhängig, schmal; eher Kuriosität |
| **S3** Ziel | AVRCP Browsing | **echte Liste**, iDrive-UI, Dreh-Drück | Phase −1 positiv; Target-Stack mit Browsing (F3) |

Bei S3 entfallen die meisten S1-Krücken („Zurueck“-Eintrag, Blindstellen bei Info-Knoten, Senderlisten nur durchklickbar). **S3 ist der strategische Grund, das Gateway zu verfolgen** — neben Stabilität (BlueZ-Entkopplung).

Was auch bei S3 **nicht** geht: Cover-Art ohne BIP; Freitext; Aktionen ohne Wiedergabe-Erwartung nach `PlayItem` müssen als Medienelemente modelliert werden (Designfrage **A18**).

---

## 4. Stack-Matrix (Host auf dem ESP)

Bezug: A17 in [OFFENE-PUNKTE.md](OFFENE-PUNKTE.md). Entscheidung **nach** Phase −1.

| Option | Metadaten (S1) | Browsing (S3) | Aufwand | Risiko |
|--------|----------------|---------------|---------|--------|
| Bluedroid unverändert | ⛔ keine TG-API (F1) | ⛔ (F2) | gering | Display leer → Projektziel verfehlt |
| Bluedroid + Patch `bta_av_act.c` | ⚠ per IDF-Fork | ⛔ | mittel | Fork bei jedem IDF-Update; S3 verbaut |
| **BTstack auf ESP32** | ✅ `avrcp_target_set_now_playing_info` | ✅ `avrcp_browsing_target_*` | höher | Controller-/Coexistence-Eigenheiten des ESP32-Ports; Lizenz nicht-kommerziell frei |

**Empfehlung (Vorlage):** BTstack, sobald Phase −1 zeigt, dass Metadaten im Fahrzeug ankommen — und zwingend, falls Browsing möglich ist. Build-System bleibt ESP-IDF (A1); nur der BT-Host-Stack wird aus A1 herausgelöst.

Zusätzlich **F4:** A2DP Sink+Source gleichzeitig auf einem ESP32 ist ausgeschlossen (Espressif, beide A2DP-Beispiele). Multi-Source (Handy/Tablet) → **Pi als A2DP-Sink** (Weg F in Betriebsmodi), nicht Relay im ESP.

---

## 5. Offene Messfragen

Vollständiger Plan: [PHASE-0-MESSPLAN.md](PHASE-0-MESSPLAN.md). Kurz:

| # | Frage | Blockiert |
|---|-------|-----------|
| M1 | Öffnet NBT Evo den Browsing-Kanal? (`SetBrowsedPlayer` / `GetFolderItems`) | S3, A17, A18 |
| M2 | Welche SDP-Feature-Bits (59 Browsing, 60 Searching, 65 NowPlaying)? | S3-Scope |
| M3 | Kommen die drei Textzeilen (Metadata) im Fahrzeug an? | S1-Minimum, Stack-Druck |
| M4 | Welcher AVRCP-Eingang wird bedient (Pass-Through-Subset)? | Mapping (Phase 0) |
| M5 | Coexistence mit gewähltem Stack (nach A17) | alles Weitere |

Durchführung Phase −1: `pidrive` (G1/G2, `tools/bmw_avrcp_probe.sh`). Hier nur Entscheidungswirkung dokumentieren.

---

## 6. Bezug

- Auftrag & Befunde: [AUFTRAG-CURSOR-2.md](AUFTRAG-CURSOR-2.md)  
- Messplan: [PHASE-0-MESSPLAN.md](PHASE-0-MESSPLAN.md)  
- Pflichtenheft §2.9 / §2.10 / §2.24  
- Offene Punkte A17–A20, R18–R23  
- Fahrzeug: `pidrive/iDriveBt.md`, `pidrive/RUNTIME_FLOWS.md`  
