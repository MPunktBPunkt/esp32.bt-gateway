# Betriebsmodi, PiDrive-Stabilität & Mehr-Clients

**Stand:** 2026-09-15 (V1.2)  
**Kontext:** Eigentümer von `pidrive` **und** `esp32.bt-gateway` — beide Repos dürfen bewusst zusammenwachsen.  
**Fragen:** (1) Wie macht der ESP PiDrive stabiler? (2) Musik vom Handy o. Ä. an den Gateway? (3) Gleichzeitig WiFi-Client + SoftAP?

---

## 1. Wie der ESP PiDrive stabiler macht

Heute hängt „Ton im Auto“ an einer Kette, die auf dem Pi **alles** ist: Quellen + PipeWire + BlueZ + AVRCP + MPRIS. Ein Fehler in D-Bus/WirePlumber/`ReserveDevice1` legt oft die ganze BMW-Strecke lahm (`BluetoothError.md`).

### 1.1 Trennung der Fehlerdomänen

| Domäne | Vorher (nur Pi) | Mit Gateway |
|--------|-----------------|-------------|
| Quellen (DAB, Web, Spotify) | Crash/Bug → oft auch BT tot | Crash → ESP hält BMW-Link, Stille/Status; Quelle neu startbar |
| PipeWire / Sink-Wahl | eng mit `bluez_output.*` verkoppelt | nur noch lokaler Capture-Sink → Gateway |
| BlueZ A2DP/AVRCP zum BMW | fragilste Schicht | **entfällt** im Gateway-Pfad |
| BMW Connect/Zündung | vermischt mit Audio-Stack | eigene ESP-State-Machine + Backoff |
| Diagnose | gemischte Logs | ESP-Analyzer + Hub-KPIs + Pi-Logs getrennt |

**Stabilitätsgewinn = weniger bewegliche Teile auf dem kritischen BMW-Pfad**, nicht „ESP ist magisch robuster als BlueZ“ — aber Classic-BT auf dedizierter Hardware mit einer Aufgabe ist einfacher zu härten als BlueZ+PipeWire+D-Bus.

### 1.2 Konkrete PiDrive-Vorteile (weil du beide Repos besitzt)

1. **`audio_output=gateway` als Default im Auto** — BlueZ nur noch Fallback/`bt` für Kopfhörer-Tests.  
2. **BMW-Bonding nur noch auf dem ESP** — Pi muss keinen CSR-Dongle mehr fürs Auto pflegen; weniger `bt_watcher`-Recovery, die Bluetooth neu startet und Nebenwirkungen hat.  
3. **Gemeinsamer PDAP-Contract** — versioniert in beiden Repos; Breaking Changes kontrolliert.  
4. **Unabhängiger Reconnect:** Zündung aus/an → ESP reconnectet BMW, ohne dass `pidrive_core` BlueZ anfassen muss.  
5. **Graceful Degradation:** Pi offline / WiFi weg → ESP meldet `GATEWAY_AUDIO_LOST`, hält BT optional, spielt Silence; PiDrive entscheidet Fallback (Klinke/HDMI/Stop) — klarer als „A2DP-Profil unavailable“.  
6. **Hub-Sichtbarkeit** — Gateway-KPIs (Buffer, Disconnect-Reason) in `iobroker.esp-hub`, unabhängig vom Pi-Debug.  
7. **Weniger Dual-AVRCP-Chaos** — ein Target (ESP); `map_event` bleibt die einzige Semantik-Quelle auf dem Pi.

### 1.3 Was PiDrive dadurch *nicht* automatisch gewinnt

- DAB Direct-ALSA bleibt ein eigenes Stabilitäts-/Routing-Thema.  
- Quellen-Bugs (welle-cli, librespot) bleiben Quellen-Bugs.  
- WiFi-Ausfall Pi↔ESP ist eine **neue** Fehlerklasse — dafür Heartbeat + klare UX.

### 1.4 Empfohlene Produkt-Story

```
PiDrive = Infotainment-Gehirn (Quellen, Menü, Trigger, Meta)
ESP     = Autofunk-Modem (nur BMW-BT + PDAP + Hub)
```

Stabilität misst sich an: weniger manuelle `bluetooth`/`wireplumber`-Restarts, längere Fahrten ohne Dropout, reproduzierbare Analyzer-Logs.

---

## 2. Musik vom Handy (und anderen Geräten) an den Gateway

Der ESP ist absichtlich **kein Mixer** und in V1 **kein** A2DP-Sink+Source-Relay. Mehrere sinnvolle Wege:

### 2.1 Wege im Überblick

| Weg | Pfad | V1? | Bemerkung |
|-----|------|-----|-----------|
| **A. Über PiDrive** | Handy → Spotify Connect / Datei / Cast auf den Pi → PDAP → ESP → BMW | **ja (Hauptweg)** | Schon heute Produktlogik; Gateway macht nur die BT-Strecke stabiler |
| **B. PDAP-Direktclient** | Handy/Laptop-App → PDAP (PCM+Meta) → ESP → BMW | **ja (Referenz: Laptop)** | Gleicher Contract wie PiDrive; Phone-App später |
| **C. SoftAP + PDAP** | Handy joined ESP-AP → PDAP | **V1.1** | Unterwegs ohne Heim-WLAN |
| **D. Browser/Web-Sender** | Handy öffnet ESP-WebUI, sendet Audio (WebAudio/MediaCapture) | später / Experiment | Bequem, aber Format/Latenz/Browser-Limits |
| **E. A2DP-Relay** | Handy —BT→ ESP (Sink) —BT→ BMW (Source) | **nicht im ESP, dauerhaft** | F4: Sink+Source gleichzeitig auf ESP32 ausgeschlossen; kein V1-Aufschub, sondern Architekturgrenze |
| **F. Pi als A2DP-Sink** | Handy/Tablet —BT→ **Pi** (Sink) → PW Capture → PDAP → ESP → BMW | **Hauptweg für Fremdgeräte** | CSR-Dongle wird frei, sobald BMW am ESP hängt; Umschaltung über PiDrive-Menü; Pi = AVRCP-Controller zum Handy (A19) |

**Empfehlung:** Stabilität und UX zuerst über **A**. Fremdgeräte-BT über **F**. Parallel **B** (Laptop-Tester) so bauen, dass eine spätere Phone-App derselbe Client ist. **E** nicht einplanen.

### 2.2 Wer „Quelle“ ist — eine Session-Regel

ESP akzeptiert **einen** aktiven PDAP-Audio-Client gleichzeitig (wie ein A2DP-Source-Gerät nur einen Stream hat).

```
Policy V1:
  - Erster authentifizierter Client, der AUDIO_START sendet, besitzt die Session
  - Zweiter Client → ERROR busy (oder Priority: PiDrive > Gast)
  - AVRCP-Events gehen immer an den Session-Owner (oder fest an PiDrive, wenn gebunden)
```

Wenn PiDrive + Handy beide „steuern“ wollen: **PiDrive bleibt Dispatcher** (Handy nur als Spotify-Quelle auf dem Pi) — einfachste UX.

### 2.3 Handy konkret (ohne Relay)

1. **Spotify/Apple Music auf dem Pi** (Connect / ähnlich) → Gateway — null Extra-Firmware.  
2. **Kleine Android/iOS-App oder Python auf dem Phone** als PDAP-Client (wie `pdap_tester`) — wenn du später „Pi aus, nur Handy→BMW“ willst.  
3. **Nicht:** Handy mit BMW-BT koppeln und ESP dazwischenschalten (Weg E).  
4. **Fremdgerät per Classic-BT:** an den **Pi** koppeln (Weg F), nicht an den ESP.

---

## 3. WiFi: Client und SoftAP gleichzeitig?

### 3.1 Kurze Antwort

**Ja, der klassische ESP32 kann `WIFI_MODE_APSTA` (Station + SoftAP).**  
**Aber:** ein Radio, gemeinsamer Kanal, geteilte Luftzeit — und parallel noch **Classic BT (A2DP)**. Das ist der enge Teil, nicht die API.

### 3.2 Technische Realität

| Aspekt | Fakt |
|--------|------|
| APSTA | ESP-IDF unterstützt SoftAP + STA gleichzeitig |
| Kanal | SoftAP folgt typisch dem Kanal des STA-AP (wenn STA verbunden) |
| Durchsatz | Stark geteilt; für PDAP-PCM + Hub-Heartbeat meist ok, für zwei schwere Streams riskant |
| Classic BT | Coexistence-Gate muss **APSTA+A2DP** extra messen, nicht nur STA+A2DP |
| Unterwegs | Ohne Fremd-AP gibt es nichts für STA — dann nur SoftAP sinnvoll |

### 3.3 Empfohlene Betriebsmodi (Policy, nicht „immer alles an“)

| Situation | WiFi-Modus | Wer spricht mit wem |
|-----------|------------|---------------------|
| **Carport / zu Hause** | **STA** → Heim-WLAN | PiDrive + Hub im LAN; SoftAP aus (oder nur Setup) |
| **Auto, Pi als Hotspot** | **STA** → Pi-AP | Stabilstes Auto-Muster: Pi = Router, ESP = Client |
| **Auto, kein Pi-Netz / nur Handy** | **SoftAP** | Handy/Laptop joinen `bt-gateway-xxxx`; PDAP direkt |
| **APSTA** | STA + SoftAP | Nur wenn Gate grün: z. B. STA Heim + SoftAP für lokales Setup-Handy; oder STA Phone-Hotspot + SoftAP für zweiten Client (selten nötig) |

```
                    ┌─ HOME:  ESP = STA ──────── Heim-Router ── Hub / Pi
BMW ←A2DP─ ESP ─────┤
                    └─ CAR:   ESP = STA → Pi-Hotspot
                       oder   ESP = SoftAP ← Handy / Pi als Client
```

**„Unterwegs Client und Server gleichzeitig“:** technisch möglich (z. B. STA am Handy-Hotspot + SoftAP für den Pi), aber meist **unnötig** — ein Netz reicht. Lieber **einen** klaren Modus wählen als dauerhaft APSTA+A2DP.

### 3.4 Config-Vorschlag

```
wifi_mode = auto | sta | ap | apsta
wifi_sta_ssid / wifi_sta_pass
wifi_ap_ssid = "bt-gateway-<mac4>" / pass
wifi_ap_when = never | setup_only | no_sta | always
```

`auto`-Logik: bekanntes STA-Netz → STA; sonst SoftAP; APSTA nur wenn explizit oder „STA verbunden und `ap_when=always`“.

### 3.5 Coexistence-Gate erweitern

Zusätzlich zu „STA + A2DP ≥ 30 min“:

- SoftAP + A2DP ≥ 30 min (PDAP vom Phone über SoftAP)  
- optional APSTA + A2DP (nur wenn Modus gewünscht)

Wenn SoftAP+A2DP fällt, Plan B: Pi immer Hotspot, ESP nur STA — SoftAP nur für Setup ohne BT-Stream.

---

## 4. Zielbild (beide Repos)

```
[ Spotify / Handy-App / DAB / Web ]
              │
              ▼
         PiDrive  ←────────── optional: Handy nur als Quelle am Pi
              │ PDAP
              ▼
     esp32.bt-gateway  ←──── optional: Laptop/Handy als PDAP-Gast (eine Session)
              │ A2DP+AVRCP
              ▼
            BMW

Programmierung: iobroker.esp-hub (USB/OTA), unabhängig vom Auto-Modus (wenn STA erreichbar).
```

**V1 Fokus:** PiDrive→ESP→BMW stabil + Hub + SoftAP zumindest für Setup.  
**V1.1:** SoftAP-PDAP für Handy/Laptop unterwegs.  
**Geplant (Pi-Seite):** Weg F — Handy/Tablet als A2DP-Sink am Pi ([OFFENE-PUNKTE.md](OFFENE-PUNKTE.md) A19).  
**Später:** native Phone-App als PDAP-Client.  
**Dauerhaft nicht im ESP:** BT-Relay Handy→ESP→BMW (Weg E / F4).

---

## 5. Offene Entscheidungen (siehe auch OFFENE-PUNKTE)

- **A10** WiFi-Default-Policy: `sta` / `ap` / `auto` (Empfehlung: `auto` = STA wenn bekannt, sonst AP; APSTA opt-in)  
- **A11** Zweiter PDAP-Client (Handy): V1 busy-reject vs. Priority PiDrive  
- **A12** Session-Owner für AVRCP: immer PiDrive wenn online, sonst Gast-Client  
- **A19** Multi-Source über Pi-Sink (Weg F) — Event-Contract / `map_event`

---

## 6. Bezug zu anderen Docs

- PiDrive-Code-Anker: [PIDRIVE-INTEGRATION.md](PIDRIVE-INTEGRATION.md)  
- Hub: [HUB-INTEGRATION.md](HUB-INTEGRATION.md)  
- Nicht-Ziel Relay / Weg F: Pflichtenheft §2.2  
- Coexistence: Pflichtenheft §2.4 + Gate inkl. SoftAP  
- AVRCP/Menü: [AVRCP-MOEGLICHKEITEN.md](AVRCP-MOEGLICHKEITEN.md)
