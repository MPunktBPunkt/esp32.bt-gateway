# Konzept — PiDrive Bluetooth Gateway (ESP32.BT-Gateway)

**Dokumentstatus:** Entwurf V1.0 (Planung)  
**Repo:** `esp32.bt-gateway`

---

## 1.1 Ausgangslage und Problem

PiDrive ist ein ausgereiftes Raspberry-Pi-basiertes Infotainment-System für BMW iDrive (getestet u. a. an F20/F21 LCI mit NBT Evo). Es beherrscht bereits:

- DAB+ (RTL-SDR + welle-cli)
- Webradio
- FM / Scanner
- Spotify Connect
- lokale Musik
- Trigger-System (Lenkrad, WebUI, CLI)
- Metadaten (MPRIS2)
- Audio-Routing über PipeWire

Das aktuelle Problem liegt nicht primär bei den Quellen, sondern in der **Bluetooth-Kette auf dem Pi**:

```
Quellen → PipeWire/WirePlumber → BlueZ → A2DP Source + AVRCP Target → BMW
```

Diese Kette ist komplex, fehleranfällig und schwer diagnostizierbar. Dokumentierte Probleme (u. a. D-Bus-/PipeWire-/ALSA-Interaktionen, die zu `br-connection-profile-unavailable` führen) entstehen genau hier. Der Raspberry Pi muss gleichzeitig Audioquelle, Bluetooth-Host, AVRCP-Target, PipeWire-System und D-Bus-Integration sein – das ist zu viel Verantwortung an einer Stelle.

## 1.2 Kernidee

Wir ziehen die gesamte Bluetooth-Classic-Verantwortung aus dem Raspberry Pi heraus und verlagern sie auf einen **dedizierten klassischen ESP32**.

Der ESP32 wird zur **Fahrzeug-Bluetooth-Hardware**:

- Er spricht ausschließlich die Sprache des BMW (A2DP Source + AVRCP Target).
- Der Raspberry Pi liefert nur noch PCM-Audio und Metadaten/Steuerbefehle über WLAN.
- Der Pi kennt im Gateway-Pfad kein BlueZ, kein A2DP und kein AVRCP mehr.

**Ergebnis:** Klare Trennung der Verantwortlichkeiten.

```
PiDrive (Infotainment-Logik)
        │
        │  PCM + Metadata + Commands   (WLAN / PDAP)
        ▼
ESP32 Gateway (Bluetooth-Hardware)
        │
        │  A2DP Source + AVRCP Target
        ▼
BMW iDrive (A2DP Sink + AVRCP Controller)
```

## 1.3 Warum genau diese Architektur?

| Vorteil | Begründung |
|---------|------------|
| **Stabilität** | Die fragilste Schicht (BlueZ + PipeWire + D-Bus) verschwindet aus dem kritischen Pfad. |
| **Diagnostizierbarkeit** | Jede Stufe (DAB → PCM → WLAN → ESP → A2DP → BMW) kann isoliert geprüft werden. |
| **Wartbarkeit** | PiDrive bleibt fast unverändert. Der ESP ist ein eigenständiges, wiederverwendbares Projekt. |
| **Beobachtbarkeit** | Der ESP kann gleichzeitig als BMW-Bluetooth-Analyzer dienen („observe first“). |
| **Zukunftssicherheit** | Andere Clients (PC, anderer ESP, zukünftige Quellen) können denselben Gateway nutzen. |
| **Entwicklungsrisiko** | BlueZ bleibt zunächst parallel erhalten → kein Big-Bang. |

## 1.4 Was der ESP bewusst **nicht** ist

- Kein zweites Infotainment-System
- Kein Audio-Mixer
- Kein Quellen-Umschalter
- Kein Handy-Relay (A2DP Sink + Source gleichzeitig)
- Kein generischer Multi-Device-Bluetooth-Adapter
- Kein Ersatz für die bestehende AUX-/HDMI-Route

Er übersetzt **ausschließlich** Client-Audio und -Steuerung in die Bluetooth-Sprache der Sink (zuerst BMW).

## 1.5 Zentrale Designregeln

1. Der ESP weiß nichts über DAB, Spotify, Sender, Menüs oder PiDrive-Geschäftslogik.
2. Der Pi weiß nichts über A2DP, SBC, AVRCP oder BlueZ (im Gateway-Pfad).
3. Alles, was der BMW sendet, wird zuerst nur beobachtet und geloggt.
4. WiFi + Classic-BT-Coexistence ist ein hartes Gate-Kriterium.
5. BlueZ bleibt als Fallback, bis der Gateway-Pfad im Auto verifiziert ist.
6. Buffer und SBC-Parameter sind parametrisierbar und werden empirisch bestimmt.
7. „BMW disconnected“ ist ein normaler Zustand, kein Fehler.

## 1.6 Entwicklungsphilosophie

- **Phase 0 zuerst:** BMW-Profil-Discovery und Analyzer mit dem ESP, bevor die volle Audio-Pipeline gebaut wird.
- **Observe first:** AVRCP und Metadata werden zunächst nur protokolliert.
- **Parallelbetrieb:** `audio backend = bluez | gateway`.
- **Empirie vor Festschreibung:** Buffer-Größen, Bitpool, Timing werden gemessen, nicht spekuliert.
- **Kleine, testbare Schritte** mit klaren Exit-Kriterien.

## 1.7 Namensgebung

| Name | Bewertung |
|------|-----------|
| **esp32.bt-gateway** | Favorit — klar, passt zu `esp32.heartrate`, `esp32.ergo`, … |
| esp32.bluetooth-gateway | eindeutig, etwas sperrig |
| esp32.bt-bridge | „Bridge“ suggeriert Sink→Source-Relay — missverständlich |
| esp32.bmw-gateway / esp32.a2dp-gateway | zu eng / zu spezifisch |

BMW-spezifische Logik intern als Profil (z. B. `BMW_NBT_EVO`), nicht im Repo-Namen.
