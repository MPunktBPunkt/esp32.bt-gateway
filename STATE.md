# STATE — esp32.bt-gateway

Lebender Projektstand. Kurz halten; Details leben in `docs/planung/`.

| Feld | Wert |
|------|------|
| Stand | 2026-09-14 |
| Phase | Planung V1.1 (Review nachgezogen) |
| Repo | `MPunktBPunkt/esp32.bt-gateway` |
| Firmware | nicht gestartet |
| Hardware | klassischer ESP32 (WROOM; Plan B WROVER) |
| Coexistence-Gate | STA+A2DP und SoftAP+A2DP |
| ESP-Hub | Pflicht (USB/OTA/Register) |
| PiDrive | Anker v0.11.127; Integration spezifiziert |

## Aktueller Fokus

- A1–A16 bestätigen oder vertagen
- Danach: `ZUSTANDSAUTOMAT.md`, `PDAP.md`, `FREERTOS.md` + Phase‑0-Skeleton

## Blocker / Risiken

1. WiFi + Classic-BT Coexistence  
2. BMW AVRCP/Metadata unvermessen  
3. RAM/CPU WROOM; A2DP 44.1 vs 48  
4. Pi PCM-Contract / DAB Direct-ALSA  

## Letzte Änderung

- Planungs-Review V1.1 (`REVIEW-V1.1.md`); Lücken Discovery/Migration/Latenz/RAM nachgezogen; Push Planung
