# esp32.bt-gateway

WiFi → Bluetooth-Classic-Gateway für ESP32 (klassisch / WROOM-32).

Der ESP32 übernimmt A2DP Source + AVRCP Target (ggf. Browsing); ein Client (z. B. PiDrive) liefert PCM-Audio, Metadaten, Steuerbefehle und optional Menübaum über WLAN (PDAP). Erste Zielanwendung: PiDrive → BMW iDrive (NBT Evo). Technisch bewusst **nicht** BMW-spezifisch benannt.

```
Client (PiDrive / Laptop / …)
        │  PCM + Metadata + Commands (+ Menu)   (WLAN / PDAP)
        ▼
ESP32 Gateway (Bluetooth-Hardware)
        │  A2DP Source + AVRCP Target (+ optional Browsing)
        ▼
Sink (BMW / Kopfhörer / …)
```

## Status

**Phase:** Planung V1.2 (BT-Steuerung analysiert, Host-Stack offen, Phase −1 ausstehend)  
**Firmware:** noch nicht gestartet  
**Lebender Stand:** [`STATE.md`](STATE.md)  
**Planung:** [`docs/planung/`](docs/planung/) — Konzept, Pflichtenheft, AVRCP-Möglichkeiten, Messplan, Hub, Betriebsmodi, Reviews

## Was dieses Projekt ist

- Dedizierte Bluetooth-Classic-Hardware (A2DP Source, AVRCP Target)
- Chance auf echtes iDrive-Listenmenü (AVRCP Browsing, Stufe S3) — messbar in Phase −1
- Analyzer-first: BMW-Verhalten zuerst vermessen, dann mappen
- Eigenständiges ESP-IDF-Projekt + minimale Client-Integration (BT-Host-Stack: Entscheidung A17)
- **ESP-Hub-Familie:** USB-Flash und OTA über [`iobroker.esp-hub`](https://github.com/MPunktBPunkt/iobroker.esp-hub) (dünner IDF-Client, kein Arduino-Compile)

## Was es bewusst nicht ist

- Kein zweites Infotainment, kein Mixer, kein Quellen-Umschalter
- Kein Handy-**BT**-Relay im ESP (Sink+Source gleichzeitig unmöglich); Multi-Source über Pi-A2DP-Sink oder PDAP/WLAN
- Kein ESP32-S3 / BLE-only
- Kein Arduino-`esp-hub-base`-Klon (nur gleicher Hub-HTTP-Contract)

## Hardware (V1)

- Klassischer ESP32 (WROOM-32 / DevKit; Plan B WROVER/PSRAM)
- Antenne 2,4 GHz
- Stabile 5‑V-Versorgung (Auto-tauglich)

## Nächste Schritte

1. Phase −1 am Pi (Browsing/Metadata) → [`docs/planung/PHASE-0-MESSPLAN.md`](docs/planung/PHASE-0-MESSPLAN.md)
2. A17 Host-Stack entscheiden → [`docs/planung/OFFENE-PUNKTE.md`](docs/planung/OFFENE-PUNKTE.md)
3. Danach: Zustandsautomat + PDAP + Phase‑0-Firmware

## Lizenz

GPL-3.0 — siehe [`LICENSE`](LICENSE). BTstack-Lizenz (falls A17 = BTstack) gesondert in README/LICENSE vermerken (R22).
