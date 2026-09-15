# STATE — esp32.bt-gateway

Lebender Projektstand. Kurz halten; Details leben in `docs/planung/`.

| Feld | Wert |
|------|------|
| Stand | 2026-09-15 |
| Phase | Planung **V2.0+** (Auftrag 4) — Teil A verbindlich; Stack/Messungen offen |
| Repo | `MPunktBPunkt/esp32.bt-gateway` |
| Firmware | nicht gestartet (A17 offen; Q12 Build-Host) |
| Hardware | klassischer ESP32 (WROOM; Plan B WROVER); Flash Dual-OTA 4 MB |
| Coexistence-Gate | STA+A2DP und SoftAP+A2DP (stackspezifisch nach A17) |
| ESP-Hub | Pflicht; Contract v0.5.12 verifiziert (A9) |
| PiDrive | Docs-Pfade gemappt; Gegenbefunde P-F1–P-F5 verankert (P12–P15) |

## Aktueller Fokus

- **Q12:** Build-Host für Flash-Budget-Messung (längster Pfad)
- Phase −1 am Pi — blockiert A17/A18
- Q1–Q13 ([AUFTRAG-CURSOR-2.md](docs/planung/AUFTRAG-CURSOR-2.md) … [4](docs/planung/AUFTRAG-CURSOR-4.md))
- Danach: A17 → Komponentenstruktur → Phase‑0-Skeleton

## Blocker / Risiken

1. Keine ESP32-Toolchain auf Planungs-Host (Q12) → A17 blockiert
2. Phase −1 / Browsing ungemessen (R19)
3. Pi-Metadaten-Auslöser falsch (P-F1 / R26) — P12 vor Abnahme
4. Menübaum ≈ 98 KB ohne Paging (P-F5 / R27)
5. Hub im Auto unerreichbar (R24); Flash-Budget ungeprüft (R25)
6. Coexistence / Stack / DAB-PW-Blockade (P-F2)

## Letzte Änderung

- Auftrag Cursor-4 + Owner-Praxis: Pairing-Bestätigung via WebUI (A4/R28); Pfad-Mapping; Flash-Proxy; P-F1–P-F5
