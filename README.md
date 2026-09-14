# esp32.bt-gateway

WiFi → Bluetooth-Classic-Gateway für ESP32 (klassisch / WROOM-32).

Der ESP32 übernimmt A2DP Source + AVRCP Target; ein Client (z. B. PiDrive) liefert PCM-Audio, Metadaten und Steuerbefehle über WLAN (PDAP). Erste Zielanwendung: PiDrive → BMW iDrive (NBT Evo). Technisch bewusst **nicht** BMW-spezifisch benannt.

```
Client (PiDrive / Laptop / …)
        │  PCM + Metadata + Commands   (WLAN / PDAP)
        ▼
ESP32 Gateway (Bluetooth-Hardware)
        │  A2DP Source + AVRCP Target
        ▼
Sink (BMW / Kopfhörer / …)
```

## Status

**Phase:** Planung V1.1 (Review)  
**Firmware:** noch nicht gestartet  
**Lebender Stand:** [`STATE.md`](STATE.md)  
**Planung:** [`docs/planung/`](docs/planung/) — inkl. PiDrive-, Hub-, Betriebsmodi- und Review-Docs

## Was dieses Projekt ist

- Dedizierte Bluetooth-Classic-Hardware (A2DP Source, AVRCP Target)
- Analyzer-first: BMW-Verhalten zuerst vermessen, dann mappen
- Eigenständiges ESP-IDF-Projekt + minimale Client-Integration
- **ESP-Hub-Familie:** USB-Flash und OTA über [`iobroker.esp-hub`](https://github.com/MPunktBPunkt/iobroker.esp-hub) (dünner IDF-Client, kein Arduino-Compile)

## Was es bewusst nicht ist

- Kein zweites Infotainment, kein Mixer, kein Quellen-Umschalter
- Kein Handy-**BT**-Relay (A2DP Sink + Source gleichzeitig); Handy über PiDrive oder PDAP/WLAN
- Kein ESP32-S3 / BLE-only
- Kein Arduino-`esp-hub-base`-Klon (nur gleicher Hub-HTTP-Contract)

## Hardware (V1)

- Klassischer ESP32 (WROOM-32 / DevKit)
- Antenne 2,4 GHz
- Stabile 5‑V-Versorgung (Auto-tauglich)

## Nächste Schritte

1. Offene Planungsfragen klären → [`docs/planung/OFFENE-PUNKTE.md`](docs/planung/OFFENE-PUNKTE.md)
2. Zustandsübergänge + PDAP-Header + Task-Prioritäten ausarbeiten
3. Phase‑0-Firmware: Analyzer + A2DP-Testton + Coexistence-Gate

## Lizenz

GPL-3.0 — siehe [`LICENSE`](LICENSE).
