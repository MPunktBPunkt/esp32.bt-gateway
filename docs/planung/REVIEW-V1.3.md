# Planungs-Review — Nachzüge V1.3 (Hub-Contract & Flash-Budget)

**Stand:** 2026-09-15  
**Anlass:** [AUFTRAG-CURSOR-3.md](AUFTRAG-CURSOR-3.md) — Hub gegen `main.js` verifiziert, Partitionen/OTA-Defer/Feld-Update.  
**Vorbild:** [REVIEW-V1.2.md](REVIEW-V1.2.md) für V1.2.

Kernkonzept unverändert: **ESP = Autofunk-Modem, PiDrive = Gehirn, Hub = Programmieren/Inventar (Heim).** Neu: Flash-Budget als A17-Kriterium und zweistufiges Update (Hub + Pi-Depot).

---

## Was an V1.2 tragfähig blieb

- Phase −1 vor Stack; S3 als strategischer Grund; A17–A20; Weg F  
- Keine Firmware vor A17  
- Pflichtenheft-Menü §2.24, AVRCP-/Messplan-Dokumente  

---

## Befunde H-F1–H-F9

Quelle: `iobroker.esp-hub/main.js` **v0.5.12** (Auftrag bezog sich auf v0.5.8; OTA/Flash/chipModel unverändert, **H-F4 korrigiert**).

| # | Kurz | Verankert |
|---|------|-----------|
| **H-F1** | OTA = Pull über HTTP aus `otaUrl` in Heartbeat-Antwort | §2.12a; HUB-INTEGRATION |
| **H-F2** | `otaUrl` einmalig → sofort NVS, sonst Update verloren | §2.12a OTA-Maschine; R11 |
| **H-F3** | `/api/ota/check` liest ohne Löschen | HUB-INTEGRATION |
| **H-F4*** | Name: ab v0.5.12 jedes nicht-leere `name` (nicht nur Erst-Heartbeat) | §2.12a Hinweis; A15/A20 bleiben relevant |
| **H-F5** | `chipModel` = Familien-Sperre | §2.12a Muss-Inventar |
| **H-F6** | USB `write_flash` Default `0x0` → Merged ohne Adapteränderung | A9 bestätigt |
| **H-F7** | Flash-Balken `/ 1966080` → Slot `0x1E0000` | Partitionstabelle D; FLASH-BUDGET |
| **H-F8** | `ios` ein JSON-String, UI als Liste | §2.13 Beispiel; Q9 offen |
| **H-F9** | Register ohne Auth | §2.15; Heimnetz |

\*Delta zum Auftragstext: sticky-first-name gilt in v0.5.12 nicht mehr.

**Konsequenz A9:** Umsetzungsweg IDF-Client ohne Adapteränderung — Status **bestätigt**.

---

## Weitere Nachzüge aus Auftrag 3

| Thema | Kurz | Ort |
|-------|------|-----|
| Partitionstabelle Dual-OTA 4 MB | Spezifikation, keine Datei vor A17 | §2.12a, OFFENE-PUNKTE D, FLASH-BUDGET |
| Flash-Budget / Codegröße | Drittes A17-Kriterium; &lt;20 % Eskalation | A17-Matrix, R25, FLASH-BUDGET |
| OTA-Defer überlebt Reset | NVS + `OTA_PENDING` + Rollback | §2.12a, §2.5 |
| Stufe-2-Update | Pi-Depot + `POST /ota-upload` | A21, R24, PIDRIVE-INTEGRATION §12 |
| WebUI eingebettet | kein Asset-FS | §2.12 |
| Merged löscht NVS/Bonding | Warnung Ersteinrichtung | §2.22 |
| Pflichtenheft V2.0 | Teil A/B-Marken, Historie | PFLICHTENHEFT.md |

---

## Dokument-Nachzüge (Checkliste)

| Dokument | Nachzug |
|----------|---------|
| `FLASH-BUDGET.md` | neu (Methode; Upstream-Zahlen ausstehend) |
| `OFFENE-PUNKTE.md` | A9 bestätigt; A17+Codegröße; A21; R24/R25; D Partitionen; E Freigabe |
| `PFLICHTENHEFT.md` | V2.0, Marken, §2.12a Ersatz, §2.12/§2.22 |
| `HUB-INTEGRATION.md` | H-F1–H-F9, Arbeitspakete H1–H10 |
| `PIDRIVE-INTEGRATION.md` | §12 `/ota-upload` + Depot; P11 |
| `STATE.md` / `README.md` | Planung V2.0 |
| `REVIEW-V1.3.md` | dieses Dokument |

---

## Bewusst nicht getan (Auftrag §8)

- Keine Firmware / `main/` / `components/`
- Keine Änderung an `iobroker.esp-hub` oder `pidrive`
- Keine Stack-Entscheidung A17 ohne Phase −1 + Größenmessung
- Keine geratenen `pidrive/docs/`-Pfade (Kap. 7)
- Keine zweite Freigabe-Checkliste neben Abschnitt E
- `0x1E0000` nicht vergrößert

---

## Owner-Fragen (weiterhin offen)

Q1–Q5 (Auftrag 2) · Q6–Q11 (Auftrag 3): Board/Flash, Analyzer vs. OTA, `freeSketch`-Semantik, Einzel-States, Signatur, Stufe 2 verbindlich.

---

## Nächster harter Schritt

1. **Flash-Budget** Upstream messen (Scratch, außerhalb Repo)  
2. **Phase −1** am Fahrzeug  
3. Dann **A17** (+ A21/Q11) → Komponentenstruktur / Phase-0-Skeleton
