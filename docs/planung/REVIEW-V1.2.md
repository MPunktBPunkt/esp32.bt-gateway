# Planungs-Review — Nachzüge V1.2 (BT-Steuerung & Stack)

**Stand:** 2026-09-15  
**Anlass:** [AUFTRAG-CURSOR-2.md](AUFTRAG-CURSOR-2.md) — verifizierte Befunde F1–F5 gegen Planung V1.1.  
**Vorbild:** [REVIEW-V1.1.md](REVIEW-V1.1.md) für V1.0.

Das Kernkonzept bleibt: **ESP = Autofunk-Modem, PiDrive = Gehirn, Hub = Programmieren/Inventar.** Neu: das Gateway ist auch die **einzige Chance auf echtes Listenmenü** im iDrive (S3). Unten die Befunde und wo sie verankert sind.

---

## Was an V1.1 tragfähig blieb

- Verantwortungsgrenzen, Coexistence-Gate, Observe-first, Ein-Session-Regel  
- PiDrive-Ist-Analyse inkl. DAB-Direct-ALSA  
- Hub über dünnen IDF-Client  
- Kein BT-Relay **im ESP** (jetzt als dauerhafte Grenze + Weg F präzisiert)

---

## Befunde F1–F5

### F1 — Bluedroid-TG ohne Metadaten-API

Öffentliche Target-API liefert keine Element Attributes; `GetElementAttributes` erreicht die App nicht. Nur Controller-Rolle hat Metadata-Cmds. Folge: Gateway-Display **leer**, wenn Bluedroid unverändert — Verlust der Bedienoberfläche (die 3 Zeilen *sind* das heutige Menü).

**Verankert:** Pflichtenheft §2.10 Vorwort; A17; R18; [AVRCP-MOEGLICHKEITEN.md](AVRCP-MOEGLICHKEITEN.md).

### F2 — Bluedroid ohne Browsing

`ESP_AVRC_FEAT_BROWSE` ohne API; Espressif-Issue nur Controller-Ziel. Echtes Listenmenü mit Bluedroid ausgeschlossen.

**Verankert:** §2.9 S3; A17; [AVRCP-MOEGLICHKEITEN.md](AVRCP-MOEGLICHKEITEN.md).

### F3 — BTstack kann Meta + Browsing

`avrcp_target_set_now_playing_info` / Browsing-Target-API; ESP32-Port aktiv; AVRCP 1.6.3; Lizenz nicht-kommerziell frei.

**Verankert:** A17-Empfehlung; R21/R22; Stack-Matrix in AVRCP-Doc.

### F4 — Kein Sink+Source gleichzeitig auf ESP32

Espressif-Dokumentation in beiden A2DP-Beispielen. Multi-Source → Pi als A2DP-Sink (Weg F), nicht Relay.

**Verankert:** §2.2; BETRIEBSMODI Weg E→dauerhaft / Weg F; A19; R20.

### F5 — BlueZ bewirbt Browsing, liefert es kaum

SDP wirbt Browsing; Target beantwortet nur Scope Media Player List. BMW-Versuch ist mit Pi/`btmon` messbar → **Phase −1**.

**Verankert:** [PHASE-0-MESSPLAN.md](PHASE-0-MESSPLAN.md); R19; Entscheidungswirkung A17/A18.

---

## Dokument-Nachzüge (Checkliste)

| Dokument | Nachzug |
|----------|---------|
| `AVRCP-MOEGLICHKEITEN.md` | neu |
| `PHASE-0-MESSPLAN.md` | neu (Phase −1) |
| `OFFENE-PUNKTE.md` | A1 entschärft; A17–A20; R18–R23 |
| `PFLICHTENHEFT.md` | §2.2, §2.3, §2.8, §2.9, §2.10, §2.16, §2.20, §2.24 |
| `BETRIEBSMODI.md` | Weg E/F |
| `KONZEPT.md` | §1.2/§1.3 Nutzen Menü; Designregel 1; Phase −1 |
| `STATE.md` / `README.md` | V1.2-Status |

---

## Bewusst nicht getan (Auftrag §5)

- Keine Firmware / `main/` / `components/`
- Keine byte-genauen PDAP-Structs
- Keine AVRCP→Trigger-Mapping-Tabelle
- Keine Arbeit in `pidrive`
- Keine Bluedroid-Patches vor A17

---

## Nächster harter Schritt

**Phase −1** am Fahrzeug → dann **A17** → erst dann Komponentenstruktur und Phase-0-Skeleton.
