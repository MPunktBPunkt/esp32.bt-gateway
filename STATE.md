# STATE — esp32.bt-gateway

Lebender Projektstand. Kurz halten; Details leben in `docs/planung/`.

| Feld | Wert |
|------|------|
| Stand | 2026-09-14 |
| Phase | Planung (Pflichtenheft V1.0 Entwurf) |
| Repo | `MPunktBPunkt/esp32.bt-gateway` |
| Firmware | nicht gestartet |
| Hardware | klassischer ESP32 (Ziel), noch kein festes Board |
| Coexistence-Gate | offen (Gate vor Audio-Pipeline) |
| PiDrive-Integration | spezifiziert, nicht implementiert |

## Aktueller Fokus

- Konzept + Pflichtenheft im Repo verankern
- Offene Architekturentscheidungen klären (siehe `docs/planung/OFFENE-PUNKTE.md`)
- Danach: detaillierte Zustandsübergänge, PDAP-Header, FreeRTOS-Prioritäten
- Parallel: Start Phase 0 (Analyzer + A2DP-Testton)

## Blocker / Risiken

1. **WiFi + Classic-BT Coexistence** auf dem klassischen ESP32 — hartes Gate
2. BMW NBT Evo AVRCP-/Metadata-Verhalten noch unvermessen
3. Stack-Wahl ESP-IDF Bluedroid vs. bestehende PlatformIO-Welt der anderen `esp32.*`-Projekte

## Letzte Änderung

- Repository angelegt, Planungsordner mit Konzept / Pflichtenheft / offenen Punkten
