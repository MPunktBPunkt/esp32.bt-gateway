# Planungs-Review — Lücken & Nachzüge (V1.1)

**Stand:** 2026-09-14  
**Anlass:** Gesamtkonzept noch einmal gegen PiDrive-Ist, Hub-Familie und ESP32-Classic-Realität geprüft.

Das Kernkonzept bleibt: **ESP = Autofunk-Modem, PiDrive = Gehirn, Hub = Programmieren/Inventar.** Unten die Punkte, die in V1.0 noch zu dünn oder fehlend waren — jetzt in Pflichtenheft / Offene Punkte verankert.

---

## Was schon solide war

- Klare Verantwortungsgrenzen, Observe-first, Coexistence-Gate  
- PiDrive-Anker (`audio_output`, `map_event`, MPRIS, DAB-ALSA-Falle)  
- Hub ohne Arduino-Zwang  
- Betriebsmodi STA/SoftAP, kein BT-Relay, eine PDAP-Session  

---

## Nachgezogene Lücken

### L1. Wie finden sich Pi und ESP? (Discovery)

Bisher nur implizit „WLAN“. Fehlt: Adressfindung.

| Mechanismus | V1 |
|-------------|-----|
| Statische IP / Config auf dem Pi (`gateway_host`) | **Pflicht-Minimum** |
| SoftAP + bekanntes SSID-Muster `bt-gateway-xxxx` | Setup / Auto ohne Heimnetz |
| mDNS (`bt-gateway.local`) | **empfohlen**, wenn STA im LAN |
| Hub zeigt IP (ioBroker) | Diagnose, nicht Steuerpfad |

**Nicht V1:** UPnP, Cloud-Broker.

### L2. A2DP-Sample-Rate vs. PCM-Contract

Pi liefert fest **44,1 kHz**. BMW/SBC kann **44,1 oder 48 kHz** verhandeln.

| Option | Empfehlung |
|--------|------------|
| SBC-Negotiation auf 44,1 erzwingen | zuerst versuchen |
| Falls BMW nur 48 will: ESP resample’t 44,1→48 vor SBC | Plan B, CPU-Last messen |
| Pi auf 48 umstellen | nur wenn Gate/CPU 44,1-Pfad scheitert |

Sonst: hörbare Pitch-Fehler oder Encoder-Reject — eigenes Risiko **R15**.

### L3. Migration: alter Pi-Bond am BMW

BMW speichert das bisherige Pi-/Dongle-Pairing. ESP ist ein **neues** BT-Gerät.

Checkliste:

1. Am iDrive altes „PiDrive“/Telefon austragen (oder Dummy-Pair überschreiben)  
2. ESP mit konfigurierbarem Namen pairen (Empfehlung Name: weiterhin `PiDrive` **oder** `PiDrive-GW` — A4)  
3. Pi-BlueZ darf denselben BMW nicht parallel reconnecten (A8)  

### L4. RAM/CPU auf klassischem ESP32

Bluedroid + WiFi + Jitter-Buffer + SBC + WebUI ist eng auf WROOM (520 KB SRAM).

- V1-Ziel: **WROOM-32**, Buffer eher klein starten, KPIs watchen  
- Falls Heap/Underruns: **WROVER (PSRAM)** als Plan B Hardware, ohne S3  
- Risiko **R16**

### L5. Latenz-Budget (Orientierung)

Kein HiFi-Studio, aber Lenkrad→Reaktion und Lippen-/Radio-Feeling:

| Segment | Grobe Zielgröße |
|---------|-----------------|
| PW Capture + PDAP UDP | ~20–40 ms |
| ESP Jitter-Buffer | 40–100 ms (parametrisierbar) |
| SBC + A2DP | ~20–40 ms |
| **Ende-zu-Ende** | **Ziel &lt; ~150–200 ms**; hartes Limit empirisch |

WebUI zeigt End-to-end-Schätzung (seq/timestamps), wenn PDAP-Timestamps da sind.

### L6. Auto-Strom / Zündung

- Versorgung: stabil **5 V**, Auto-tauglich (Verpol-/Lastabwurf beachten — Hardware-Hinweis, kein Schaltplan hier)  
- Verhalten: Zündung aus → BMW disconnect = `NO_BMW`, kein Alarm  
- Optional später: Ignition-GPIO / Spannungsmonitor — **nicht V1-Pflicht**  
- Brownout-Detector + Task-Watchdog: **ja (V1)**

### L7. Factory-Reset & SoftAP-Security

- WebUI (und optional GPIO-Button): NVS löschen (WiFi, Bonding, PSK)  
- SoftAP: **WPA2** mit konfigurierbarem Passwort (kein offenes AP im Feld)  
- PDAP weiter PSK im HELLO  

### L8. Log-Retention

- Ringbuffer im RAM + Export WebUI (JSON/CSV) — V1  
- Optional Append auf SPIFFS/LittleFS — wenn Platz; nicht SD-Pflicht  
- Analyzer-Logs überleben Reset nicht ohne Flash — dokumentieren  

### L9. PDAP HELLO Capabilities

Neben VERSION/AUTH:

```
capabilities: { audio_pcm_44100_s16le, metadata_v1, events_v1 }
max_buffer_ms, fw_version, bt_name
```

Damit Pi und Tester Feature-Mismatch früh erkennen (statt stiller Underruns).

### L10. Build-/Release-Disziplin

- ESP-IDF-Version pinnen (`idf.py` / `version.txt` im Repo)  
- Zwei Artefakte: merged (USB `0x0`) + app (OTA)  
- SemVer = Hub-`version` = Bin-Name  

---

## Konzept-Nachschärfung (eine Seite)

```
                    ┌── iobroker.esp-hub (USB/OTA/Inventar)
                    │
[Quellen/Handy] → PiDrive ──PDAP──► ESP32.bt-gateway ──A2DP/AVRCP──► BMW
                    │                    │
                    │                    ├── SoftAP oder STA (auto)
                    │                    └── Analyzer / WebUI
                    └── BlueZ nur Fallback (nicht parallel zum selben BMW)
```

**Stabilität** entsteht durch Domänentrennung + eine Session + erzwungenes PCM + deferred OTA — nicht durch mehr Features auf dem ESP.

**Bewusst später:** Byte-PDAP, FreeRTOS-Prios, exakte AVRCP-Tabelle, DAB-PW-Bridge, Phone-App, APSTA-Dauerbetrieb, Clock-Drift-Korrektur.

---

## Aktualisierte Offene Punkte

Neu: **A13–A16**, Risiken **R15–R17** — siehe [OFFENE-PUNKTE.md](OFFENE-PUNKTE.md).
