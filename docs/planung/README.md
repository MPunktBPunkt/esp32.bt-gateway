# `docs/planung/` — Konzept & Pflichtenheft

Planungsunterlagen für `esp32.bt-gateway` (PiDrive Bluetooth Gateway).

> **Lebender Stand:** [`../../STATE.md`](../../STATE.md)

| Datei | Inhalt |
|-------|--------|
| [AUFTRAG-CURSOR-2.md](AUFTRAG-CURSOR-2.md) | Auftrag V1.2: Befunde F1–F5, Redaktionsplan |
| [AUFTRAG-CURSOR-3.md](AUFTRAG-CURSOR-3.md) | Auftrag V2.0: Hub-Contract, Flash-Budget, OTA-Defer |
| [AUFTRAG-CURSOR-4.md](AUFTRAG-CURSOR-4.md) | Pfad-Mapping, Flash-Proxy, PiDrive-Gegenbefunde P-F1–P-F5 |
| [AVRCP-MOEGLICHKEITEN.md](AVRCP-MOEGLICHKEITEN.md) | iDrive-BT-Steuerung, Kanäle, Stufen S1–S3, Stack-Matrix |
| [PHASE-0-MESSPLAN.md](PHASE-0-MESSPLAN.md) | Phase −1 (Pi) vor Phase 0 (ESP) und Coexistence |
| [FLASH-BUDGET.md](FLASH-BUDGET.md) | Partitionen, Messmethode, Größenlimit für A17 |
| [KONZEPT.md](KONZEPT.md) | Ausgangslage, Kernidee, Architektur, Designregeln |
| [PFLICHTENHEFT.md](PFLICHTENHEFT.md) | Pflichtenheft **V2.0** — Teil A fix / Teil B Entwurf |
| [PIDRIVE-INTEGRATION.md](PIDRIVE-INTEGRATION.md) | Ist-Analyse `pidrive` @ 0.11.127 + PDAP + `/ota-upload` |
| [HUB-INTEGRATION.md](HUB-INTEGRATION.md) | iobroker.esp-hub: verifizierter Contract, USB/OTA Stufe 1+2 |
| [BETRIEBSMODI.md](BETRIEBSMODI.md) | PiDrive-Stabilität, Handy/PDAP, Weg F (Pi-Sink), WiFi |
| [REVIEW-V1.1.md](REVIEW-V1.1.md) | Planungs-Review V1.1 |
| [REVIEW-V1.2.md](REVIEW-V1.2.md) | Nachzüge F1–F5 |
| [REVIEW-V1.3.md](REVIEW-V1.3.md) | Nachzüge H-F1–H-F9, Flash, Feld-OTA |
| [REVIEW-V1.4.md](REVIEW-V1.4.md) | Pfad-Mapping, webradio-Proxy, P-F1–P-F5 |
| [OFFENE-PUNKTE.md](OFFENE-PUNKTE.md) | A1/A9/A17–A21, Q12/Q13, Risiken R24–R27, Freigabe E |

## Kurzfassung

Bluetooth-Classic (BlueZ/A2DP/AVRCP) vom Raspberry Pi auf einen dedizierten ESP32 auslagern. Der Pi spricht PCM + Metadaten + Commands (+ optional Menübaum) über WLAN; der ESP ist die Fahrzeug-Bluetooth-Hardware — und die Chance auf echtes AVRCP-Browsing (S3).

**Hub:** Gerät erscheint in `iobroker.esp-hub`, USB-Flash + OTA-Pull wie die Familie — über dünnen IDF-`HubClient`, Contract am Adaptercode verifiziert. Im Auto: lokales OTA über PiDrive-Depot. Siehe [HUB-INTEGRATION.md](HUB-INTEGRATION.md).

**PiDrive-Anker (Ist):** neuer `audio_output=gateway`, Capture von virtuellem PipeWire-Sink (44.1/S16LE/Stereo erzwingen), Reverse-AVRCP über bestehende Event-Namen/`map_event` → `/tmp/pidrive_cmd`. Produktiver Client in `pidrive`; hier nur Contract + Referenzclient. Siehe [PIDRIVE-INTEGRATION.md](PIDRIVE-INTEGRATION.md).

**Betriebsmodi:** ESP macht PiDrive stabiler durch Trennung der BMW-BT-Domäne; Handy über PiDrive/Spotify, Pi-A2DP-Sink (Weg F) oder PDAP — **kein** BT-Relay im ESP (F4). WiFi: STA/SoftAP/`APSTA`. Siehe [BETRIEBSMODI.md](BETRIEBSMODI.md).

**Entwicklungsphilosophie:** Phase −1 (Pi) + Flash-Budget → A17 → Phase 0 (ESP observe first) → Coexistence-Gate → Audio → AVRCP/Menü → PDAP → PiDrive-Integration. BlueZ bleibt parallel als Fallback zum BMW.
