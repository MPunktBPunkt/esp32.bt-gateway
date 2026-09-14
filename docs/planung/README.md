# `docs/planung/` — Konzept & Pflichtenheft

Planungsunterlagen für `esp32.bt-gateway` (PiDrive Bluetooth Gateway).

> **Lebender Stand:** [`../../STATE.md`](../../STATE.md)

| Datei | Inhalt |
|-------|--------|
| [KONZEPT.md](KONZEPT.md) | Ausgangslage, Kernidee, Architektur, Designregeln |
| [PFLICHTENHEFT.md](PFLICHTENHEFT.md) | Pflichtenheft V1.0 — implementierbare Spezifikation |
| [PIDRIVE-INTEGRATION.md](PIDRIVE-INTEGRATION.md) | Ist-Analyse `pidrive` @ 0.11.127 + konkrete Schnittstellen |
| [HUB-INTEGRATION.md](HUB-INTEGRATION.md) | iobroker.esp-hub: USB-Flash, Heartbeat, OTA (IDF-Client) |
| [BETRIEBSMODI.md](BETRIEBSMODI.md) | PiDrive-Stabilität, Handy/PDAP-Clients, WiFi STA/SoftAP/APSTA |
| [REVIEW-V1.1.md](REVIEW-V1.1.md) | Planungs-Review: Lücken (Discovery, Rate, Migration, RAM, …) |
| [OFFENE-PUNKTE.md](OFFENE-PUNKTE.md) | Entscheidungen, Risiken, empfohlene Reihenfolge |

## Kurzfassung

Bluetooth-Classic (BlueZ/A2DP/AVRCP) vom Raspberry Pi auf einen dedizierten ESP32 auslagern. Der Pi spricht nur noch PCM + Metadaten + Commands über WLAN; der ESP ist die Fahrzeug-Bluetooth-Hardware.

**Hub:** Gerät erscheint in `iobroker.esp-hub`, USB-Flash + OTA-Push wie die Familie — über dünnen IDF-`HubClient`, nicht Arduino-Basis. Compile-Tab entfällt. Siehe [HUB-INTEGRATION.md](HUB-INTEGRATION.md).

**PiDrive-Anker (Ist):** neuer `audio_output=gateway`, Capture von virtuellem PipeWire-Sink (44.1/S16LE/Stereo erzwingen), Reverse-AVRCP über bestehende Event-Namen/`map_event` → `/tmp/pidrive_cmd`. Produktiver Client in `pidrive`; hier nur Contract + Referenzclient. Siehe [PIDRIVE-INTEGRATION.md](PIDRIVE-INTEGRATION.md).

**Betriebsmodi:** ESP macht PiDrive stabiler durch Trennung der BMW-BT-Domäne; Handy-Musik primär über PiDrive (Spotify etc.) oder später PDAP-Direktclient — kein BT-Relay in V1. WiFi: STA und SoftAP sind möglich (`APSTA`), aber unterwegs meist **ein** Modus (Pi-Hotspot→ESP als STA, oder ESP-SoftAP). Siehe [BETRIEBSMODI.md](BETRIEBSMODI.md).

**Entwicklungsphilosophie:** Phase 0 (observe first) → Coexistence-Gate → Audio → AVRCP-Mapping → PDAP → PiDrive-Integration. BlueZ bleibt parallel als Fallback.
