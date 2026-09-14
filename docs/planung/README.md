# `docs/planung/` — Konzept & Pflichtenheft

Planungsunterlagen für `esp32.bt-gateway` (PiDrive Bluetooth Gateway).

> **Lebender Stand:** [`../../STATE.md`](../../STATE.md)

| Datei | Inhalt |
|-------|--------|
| [KONZEPT.md](KONZEPT.md) | Ausgangslage, Kernidee, Architektur, Designregeln |
| [PFLICHTENHEFT.md](PFLICHTENHEFT.md) | Pflichtenheft V1.0 — implementierbare Spezifikation |
| [OFFENE-PUNKTE.md](OFFENE-PUNKTE.md) | Entscheidungen, Risiken, empfohlene Reihenfolge |

## Kurzfassung

Bluetooth-Classic (BlueZ/A2DP/AVRCP) vom Raspberry Pi auf einen dedizierten ESP32 auslagern. Der Pi spricht nur noch PCM + Metadaten + Commands über WLAN; der ESP ist die Fahrzeug-Bluetooth-Hardware.

**Entwicklungsphilosophie:** Phase 0 (observe first) → Coexistence-Gate → Audio → AVRCP-Mapping → PDAP → PiDrive-Integration. BlueZ bleibt parallel als Fallback.
