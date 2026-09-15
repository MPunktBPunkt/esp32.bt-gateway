# STATE — esp32.bt-gateway

Lebender Projektstand. Kurz halten; Details leben in `docs/planung/`.

| Feld | Wert |
|------|------|
| Stand | 2026-09-15 |
| Phase | Planung V1.2 (BT-Steuerung analysiert, Stack offen, Phase −1 ausstehend) |
| Repo | `MPunktBPunkt/esp32.bt-gateway` |
| Firmware | nicht gestartet (A17 offen) |
| Hardware | klassischer ESP32 (WROOM; Plan B WROVER) |
| Coexistence-Gate | STA+A2DP und SoftAP+A2DP (stackspezifisch nach A17) |
| ESP-Hub | Pflicht (USB/OTA/Register) |
| PiDrive | Anker v0.11.127; Integration spezifiziert |

## Aktueller Fokus

- Phase −1 am Pi (Browsing/Metadata) — blockiert A17/A18
- Eigentümerfragen Q1–Q5 ([AUFTRAG-CURSOR-2.md](docs/planung/AUFTRAG-CURSOR-2.md))
- Danach: A17 → Komponentenstruktur → Phase‑0-Skeleton

## Blocker / Risiken

1. WiFi + Classic-BT Coexistence (Stack noch offen)
2. Bluedroid ohne Target-Metadaten (F1) / ohne Browsing (F2) → A17
3. NBT Evo Browsing ungemessen (R19) — Phase −1
4. BMW AVRCP/Metadata unvermessen (Phase 0)
5. RAM/CPU WROOM; A2DP 44.1 vs 48; Menübaum (R23)
6. Pi PCM-Contract / DAB Direct-ALSA

## Letzte Änderung

- Auftrag Cursor-2 umgesetzt: `AVRCP-MOEGLICHKEITEN.md`, `PHASE-0-MESSPLAN.md`, A17–A20, REVIEW-V1.2
