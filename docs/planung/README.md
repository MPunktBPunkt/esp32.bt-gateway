# `docs/planung/` — Konzept & Pflichtenheft

Planungsunterlagen für `esp32.bt-gateway` (PiDrive Bluetooth Gateway).

> **Lebender Stand:** [`../../STATE.md`](../../STATE.md)

| Datei | Inhalt |
|-------|--------|
| [AUFTRAG-CURSOR-2.md](AUFTRAG-CURSOR-2.md) | Auftrag V1.2: Befunde F1–F5, Redaktionsplan |
| [AVRCP-MOEGLICHKEITEN.md](AVRCP-MOEGLICHKEITEN.md) | iDrive-BT-Steuerung, Kanäle, Stufen S1–S3, Stack-Matrix |
| [PHASE-0-MESSPLAN.md](PHASE-0-MESSPLAN.md) | Phase −1 (Pi) vor Phase 0 (ESP) und Coexistence |
| [KONZEPT.md](KONZEPT.md) | Ausgangslage, Kernidee, Architektur, Designregeln |
| [PFLICHTENHEFT.md](PFLICHTENHEFT.md) | Pflichtenheft — implementierbare Spezifikation (V1.2) |
| [PIDRIVE-INTEGRATION.md](PIDRIVE-INTEGRATION.md) | Ist-Analyse `pidrive` @ 0.11.127 + konkrete Schnittstellen |
| [HUB-INTEGRATION.md](HUB-INTEGRATION.md) | iobroker.esp-hub: USB-Flash, Heartbeat, OTA (IDF-Client) |
| [BETRIEBSMODI.md](BETRIEBSMODI.md) | PiDrive-Stabilität, Handy/PDAP, Weg F (Pi-Sink), WiFi |
| [REVIEW-V1.1.md](REVIEW-V1.1.md) | Planungs-Review V1.1 (Discovery, Rate, Migration, RAM, …) |
| [REVIEW-V1.2.md](REVIEW-V1.2.md) | Nachzüge F1–F5 (Bluedroid-Meta, Browsing, BTstack, Multi-Source) |
| [OFFENE-PUNKTE.md](OFFENE-PUNKTE.md) | Entscheidungen A1/A17–A20, Risiken, Reihenfolge |

## Kurzfassung

Bluetooth-Classic (BlueZ/A2DP/AVRCP) vom Raspberry Pi auf einen dedizierten ESP32 auslagern. Der Pi spricht PCM + Metadaten + Commands (+ optional Menübaum) über WLAN; der ESP ist die Fahrzeug-Bluetooth-Hardware — und die Chance auf echtes AVRCP-Browsing (S3).

**Hub:** Gerät erscheint in `iobroker.esp-hub`, USB-Flash + OTA-Push wie die Familie — über dünnen IDF-`HubClient`, nicht Arduino-Basis. Compile-Tab entfällt. Siehe [HUB-INTEGRATION.md](HUB-INTEGRATION.md).

**PiDrive-Anker (Ist):** neuer `audio_output=gateway`, Capture von virtuellem PipeWire-Sink (44.1/S16LE/Stereo erzwingen), Reverse-AVRCP über bestehende Event-Namen/`map_event` → `/tmp/pidrive_cmd`. Produktiver Client in `pidrive`; hier nur Contract + Referenzclient. Siehe [PIDRIVE-INTEGRATION.md](PIDRIVE-INTEGRATION.md).

**Betriebsmodi:** ESP macht PiDrive stabiler durch Trennung der BMW-BT-Domäne; Handy über PiDrive/Spotify, Pi-A2DP-Sink (Weg F) oder PDAP — **kein** BT-Relay im ESP (F4). WiFi: STA/SoftAP/`APSTA`. Siehe [BETRIEBSMODI.md](BETRIEBSMODI.md).

**Entwicklungsphilosophie:** Phase −1 (Pi) → A17 → Phase 0 (ESP observe first) → Coexistence-Gate → Audio → AVRCP/Menü → PDAP → PiDrive-Integration. BlueZ bleibt parallel als Fallback zum BMW.
