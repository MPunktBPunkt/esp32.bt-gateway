# Planungs-Review — Nachzüge V1.4 (Pfad-Mapping & PiDrive-Gegenbefunde)

**Stand:** 2026-09-15  
**Anlass:** [AUFTRAG-CURSOR-4.md](AUFTRAG-CURSOR-4.md)  
**Vorbild:** [REVIEW-V1.3.md](REVIEW-V1.3.md)

---

## Was an V1.3 / V2.0 tragfähig blieb

- Partitionstabelle, §2.12a Hub/OTA, Teil-A/B-Marken  
- H-F4-Korrektur gegen `main.js` v0.5.12  
- Flash-Messmethode ohne Spekulation  

---

## Nachzüge Auftrag 4

| Thema | Kurz | Ort |
|-------|------|-----|
| PiDrive-Pfad-Mapping | Docs-Umzug vollzogen; neun tote Verweise korrigiert | [AUFTRAG-CURSOR-4.md](AUFTRAG-CURSOR-4.md) Kap. 1; AVRCP-/PIDRIVE-Docs |
| Flash-Proxy | `webradio` Leitwert 63 %; Band 61–69 %; Arduino pessimistisch | [FLASH-BUDGET.md](FLASH-BUDGET.md) §4.2 |
| **P-F1** | Metadaten nur bei Menü-`rev` | P12; §2.10; R26 |
| **P-F2** | DAB-Bypass unbedingt (PW-Blockade) | §2.17 Gate verschärft |
| **P-F3** | Pi nur A2DP-Source | A19 Aufwand / P15 |
| **P-F4** | `bt` nicht in `commit_source()` | P14; ZUSTANDSMASCHINE |
| **P-F5** | Menübaum ≈ 98 KB, kein Paging | §2.24 paginiert; Nicht-Ziel; R27 |
| Q12 / Q13 | Build-Host; Menü-Paginierung Timing | OFFENE-PUNKTE |
| **R28 / A4** | iDrive wartet auf Remote-Pairing-Confirm → WebUI als Handy-Dialog | A4, §2.5/`PAIRING_CONFIRM`, §2.12, §2.22 |

**Nicht verlinkt** (existieren noch nicht): `BMW-AVRCP-PROBE.md`, `BMW-DISPLAY-PROBE.md`.

---

## Bewusst nicht getan

- Keine Firmware; kein Edit in `pidrive` / `iobroker.esp-hub`  
- Keine A17; keine 8/16-MB-Eskalation auf Proxy-Basis  
- `0x1E0000` unverändert  

---

## Nächster harter Schritt

**Q12** (Build-Host) → Flash-Messung → Phase −1 → **A17**
