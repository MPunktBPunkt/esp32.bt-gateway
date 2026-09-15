# STATE — esp32.bt-gateway

Lebender Projektstand. Kurz halten; Details leben in `docs/planung/`.

| Feld | Wert |
|------|------|
| Stand | 2026-09-15 |
| Phase | Planung **V2.0** — Teil A (Hub/OTA/Partitionen) verbindlich; Teil B Entwurf |
| Repo | `MPunktBPunkt/esp32.bt-gateway` |
| Firmware | nicht gestartet (A17 offen) |
| Hardware | klassischer ESP32 (WROOM; Plan B WROVER); Flash-Spez. Dual-OTA 4 MB |
| Coexistence-Gate | STA+A2DP und SoftAP+A2DP (stackspezifisch nach A17) |
| ESP-Hub | Pflicht; Contract gegen `main.js` v0.5.12 verifiziert (A9 bestätigt) |
| PiDrive | Anker v0.11.127; Integration + Stufe-2-OTA spezifiziert |

## Aktueller Fokus

- Phase −1 am Pi (Browsing/Metadata) — blockiert A17/A18
- Flash-Budget Upstream-Messung ([FLASH-BUDGET.md](docs/planung/FLASH-BUDGET.md)) — Input A17 / R25
- Eigentümerfragen Q1–Q11 ([AUFTRAG-CURSOR-2.md](docs/planung/AUFTRAG-CURSOR-2.md), [AUFTRAG-CURSOR-3.md](docs/planung/AUFTRAG-CURSOR-3.md))
- Danach: A17 → Komponentenstruktur → Phase‑0-Skeleton

## Blocker / Risiken

1. WiFi + Classic-BT Coexistence (Stack noch offen)
2. Bluedroid ohne Target-Metadaten (F1) / ohne Browsing (F2) → A17
3. NBT Evo Browsing ungemessen (R19) — Phase −1
4. Flash-Budget ungeprüft (R25) — Upstream-Messung ausstehend
5. Hub im Auto unerreichbar (R24) — Stufe 2 / A21
6. BMW AVRCP/Metadata unvermessen (Phase 0); RAM/CPU; Pi PCM / DAB

## Letzte Änderung

- Auftrag Cursor-3: Hub-Contract H-F*, Partitionstabelle, FLASH-BUDGET, Pflichtenheft V2.0, A21/R24/R25, REVIEW-V1.3
