# PiDrive-Integration — Ist-Analyse & Schnittstellenplan

**Stand:** 2026-09-14  
**Quelle:** [MPunktBPunkt/pidrive](https://github.com/MPunktBPunkt/pidrive) @ `0.11.127`  
**Zweck:** Reale Codebasis in die Gateway-Planung einbeziehen (nicht nur Konzept-Annahmen).

---

## 1. Kurzfazit

PiDrive hat bereits klar getrennte Schichten für **Audio-Route**, **AVRCP→Trigger**, **Metadaten (MPRIS2)** und **BT-Verbindung**. Ein ESP-Gateway passt als **neuer `audio_output`-Backend** plus **`integration/gateway_client`**, der:

1. PCM von einem dedizierten PipeWire-Sink capture’t,
2. dieselben Metadaten wie `mpris2.update()` an PDAP schickt,
3. Lenkrad-Events als PDAP-Events empfängt und **dieselbe** `map_event()`-Logik bzw. `/tmp/pidrive_cmd` nutzt.

BlueZ bleibt Parallelbetrieb (`audio_output=bt`), bis der Gateway-Pfad im Auto steht.

**Keine bestehenden Gateway-/PDAP-/ESP-Hooks** im Repo — Integration ist greenfield, aber die Ankerpunkte sind klar.

---

## 2. Relevante PiDrive-Struktur

```
pidrive/
├── modules/audio.py              # decide_audio_route / set_output / get_mpv_args
├── modules/bluetooth/            # A2DP connect, agent, watcher, sinks
├── integration/avrcp_trigger.py  # BlueZ D-Bus → map_event → /tmp/pidrive_cmd
├── mpris2.py                     # BMW-Display-Metadaten + 2. AVRCP-Ingress
├── trigger/                      # Dispatcher (td_*)
├── settings.py                   # audio_output: auto|klinke|bt|hdmi
├── main_core.py                  # Core-Loop, MPRIS-Start
└── cli/                          # pidrivectl audio|bt|avrcp|…

systemd/
├── pidrive_core.service
├── pidrive_avrcp.service         # → integration/avrcp_trigger.py
├── pipewire*.service / wireplumber.service
└── pidrive-mpris2.conf

Docs (Pflichtlektüre für Gateway):
├── iDriveBt.md                   # Profile, AVRCP, SBC, Absolute Volume
├── BluetoothError.md             # br-connection-profile-unavailable Root Cause
├── ARCHITECTURE.md / RUNTIME_FLOWS.md
```

---

## 3. Ist-Zustand Bluetooth (was der ESP ersetzt)

| Profil | PiDrive heute | BMW |
|--------|---------------|-----|
| A2DP | Source via BlueZ + PipeWire (`bluez_output.<MAC>.N`) | Sink |
| AVRCP | Target via BlueZ + `avrcp_trigger` (+ MPRIS Next/Prev/…) | Controller |
| MPRIS2 | `org.mpris.MediaPlayer2.pidrive` (system bus) → BlueZ → Display | — |
| Codec | SBC only (`bluez5.codecs = [ sbc ]`) | erwartet ~44.1/48 kHz |

**Hardware-Hinweis:** Pi nutzt oft CSR-USB-Dongle (`btusb`). Gateway-Pfad macht den Dongle für die **BMW-Verbindung** überflüssig; Pi-BT kann parallel für Kopfhörer o. Ä. bleiben (Produktentscheidung).

**Dokumentierte Fragilität** (`BluetoothError.md`): A2DP-Fails waren oft D-Bus/`ReserveDevice1`/WirePlumber/MediaEndpoint — genau die Schicht, die der ESP eliminiert.

---

## 4. Audio-Routing heute → Gateway-Erweiterung

### 4.1 Policy (`modules/audio.py`)

- Setting: `audio_output` ∈ `{auto, klinke, bt, hdmi}`
- State: `/tmp/pidrive_audio_state.json` (`requested`, `effective`, `reason`, `sink`)
- CLI: `pidrivectl audio route klinke|bt|hdmi|auto` → Trigger `audio_*`

`decide_audio_route()` wählt bei `bt`/`auto` den BlueZ-Sink (`get_bt_sink()`), sonst ALSA/HDMI.

### 4.2 Geplante Erweiterung (minimal)

| Änderung | Ort |
|----------|-----|
| `audio_output: "gateway"` (+ Defaults) | `settings.py` / `config/settings.json` |
| Branch in `decide_audio_route()` / `set_output()` | `modules/audio.py` |
| Default-Sink = **virtueller/local Sink** (nicht `bluez_output.*`) | PipeWire-Node + Capture durch Client |
| CLI `audio route gateway`, `gateway status` | `cli/`, `td_hardware.py` |
| Web-Whitelist falls UI | `web/shared/constants.py` |

**Parallelbetrieb-Modell:**

```
Players → PipeWire ─┬─ audio_output=bt      → bluez_output.* → BMW (BlueZ)
                    └─ audio_output=gateway → pidrive_gateway (null/loopback)
                                              → gateway_client → PDAP PCM → ESP → BMW
```

Regel: Bei aktivem `gateway` **keine** zweite A2DP-Source zum selben BMW (sonst Doppel-Bonding / Button-Doppelteuerung).

### 4.3 Kritischer Ist-Befund: DAB bypass’t PipeWire

`modules/radio/dab_play.py` startet **welle-cli direkt auf ALSA** (`welle_direct_alsa: True`), entfernt `PULSE_*` / `PIPEWIRE_*` aus der Env und schreibt `/etc/asound.conf` auf die Klinke.

→ `audio route bt` liefert **kein DAB über Bluetooth**, solange dieser Pfad so bleibt.  
→ Für Gateway-DAB muss DAB **explizit** über Pulse/PipeWire (oder gemeinsamen Capture-Punkt) laufen. Das ist ein **eigenes PiDrive-Arbeitspaket**, nicht nur ESP-Firmware.

### 4.4 PCM-Format (Abweichung vom Pflichtenheft)

Pflichtenheft V1 nimmt fest **44,1 kHz / S16LE / Stereo** an. PiDrive-Realität:

| Quelle | Typisches Format |
|--------|------------------|
| Webradio / lokal / Spotify | decoderabhängig (oft 44.1/48), via PipeWire |
| FM / Scanner | **32 kHz mono** (rtl_fm) → mpv |
| DAB | welle-cli / ALSA — nicht im PW-Pfad |

**Konsequenz für PDAP:** Entweder

- **A)** Pi resample’t immer auf festes Gateway-Format (empfohlen: 44.1 stereo S16LE in einem PW-Converter-Node), oder  
- **B)** PDAP verhandelt Sample-Rate/Channels und ESP resample’t (schwerer, mehr CPU auf ESP).

Empfehlung: **A** — Capture-Node erzwingt Format; ESP bleibt dumm.

---

## 5. AVRCP / Trigger — wiederverwenden, nicht neu erfinden

### 5.1 Event-Namen (Ist)

Aus `integration/avrcp_trigger.py`:

| AVRCP / D-Bus | Internes Event |
|---------------|----------------|
| Next | `next` |
| Previous | `previous` |
| Play / Pause / PlayPause | `play` / `pause` / `play_pause` |
| Stop | `stop` |
| Vol± | `volumeup` / `volumedown` |
| FF / Rew | `fast_forward` / `rewind` |

Double-Tap Play innerhalb **1,2 s** → Trigger `cat:0` („Jetzt läuft“).

### 5.2 Mapping → Trigger (`map_event`)

Kontextabhängig (Auszug):

| Event | Menü | Radio FM | Radio DAB | sonst |
|-------|------|----------|-----------|-------|
| next | `down` | `fm_next` | `dab_next` | `down` |
| previous | `up` | `fm_prev` | `dab_prev` | `up` |
| play/pause | `enter` | `radio_stop` | `radio_stop` | … |
| stop | `back` | `back` | `back` | `back` |
| volume* | `vol_up` / `vol_down` | (alle Kontexte) | | |

Ingress-Ziel: Zeile in **`/tmp/pidrive_cmd`** → `trigger_dispatcher`.

### 5.3 Dual-AVRCP-Problem (heute)

1. `avrcp_trigger` — kontextsensitiv  
2. `mpris2.PiDrivePlayer` — festes Mapping (Next→`down`, …)

Dokumentiert in `iDriveBt.md`. Mit ESP als einzigem AVRCP-Target zum BMW entfällt BlueZ-AVRCP zum Auto; **gateway_client** soll PDAP-Events in die **Event-Schicht** einspeisen und **`map_event()` wiederverwenden** (eine Verhaltensquelle).

### 5.4 Absolute Volume

PiDrive fixiert MPRIS Volume auf 1.0; BMW-DSP regelt Lautstärke. Gateway muss **nicht** Absolute-Volume gegen den BMW kämpfen — nur Pass-Through von Lenkrad-Vol± als Events (wie heute).

---

## 6. Metadaten

Heute: `mpris2.update(S, menu)` → Title / Artist / Album (3 BMW-Zeilen).

Quellen: Playback-Status, ICY (`mpv_meta`), DAB-DLS (`dab_dls`), Menü-Labels.

**Gateway-Pfad:** dieselben Felder als PDAP `METADATA` schicken; ESP → AVRCP Element Attributes.  
Bei aktivem Gateway: MPRIS zum BMW optional idle (BlueZ nicht mehr Display-Owner). Status-JSON `/tmp/pidrive_status.json` bleibt Diagnose-Anker.

---

## 7. Konkrete PiDrive-Änderungsliste (Repo `pidrive`)

| # | Arbeitspaket | Dateien / Units |
|---|--------------|-----------------|
| P1 | Config: `audio_output=gateway`, `gateway_host`, `gateway_token`, … | `settings.py`, `config/settings.json` |
| P2 | Route-Policy + set_output | `modules/audio.py`, `td_hardware.py` |
| P3 | PipeWire: virtueller Sink + fester Resample auf 44.1/S16LE/Stereo | `pipewire-config/`, Installer-Snippet |
| P4 | `integration/gateway_client.py` (PDAP Control TCP + Audio UDP) | neu |
| P5 | Optional `systemd/pidrive_gateway.service` | neu |
| P6 | Reverse path: PDAP EVENT → `map_event` oder `/tmp/pidrive_cmd` | gateway_client |
| P7 | Metadata-Push aus denselben Inputs wie MPRIS | gateway_client ↔ `playback_meta` / status |
| P8 | DAB über PW/Pulse statt Direct-ALSA (wenn Gateway DAB können soll) | `dab_play.py` — **separates Risiko** |
| P9 | CLI/Web: `gateway status`, route gateway | `cli/`, Web-API |
| P10 | Bei `gateway`: onboard A2DP zum BMW idle / nicht auto-connect | `bt_connect` / `bt_watcher` Policy |

**Repo-Scope (A5):** produktiver Client bleibt in **`pidrive`**; in `esp32.bt-gateway` nur PDAP-Contract + Python-Referenzclient (`clients/pdap_tester/`).

---

## 8. PDAP ↔ PiDrive Event-Contract (V1-Skizze)

PDAP-Events vom ESP sollten die **internen Event-Namen** aus §5.1 tragen (nicht BMW-Opcodes, nicht fertige Trigger):

```
EVENT { "name": "next" | "previous" | "play_pause" | "stop" | "volumeup" | "volumedown" | …,
        "raw": { … optional Analyzer-Felder … } }
```

PiDrive: `map_event(name, ctx)` → Trigger. So bleibt Menü-/Radio-Kontext auf dem Pi.

Metadata Pi→ESP:

```
METADATA { "title", "artist", "album", "source", "playing": bool, … }
```

Status ESP→Pi: WiFi/BT/Buffer/KPIs (siehe Pflichtenheft §2.11).

---

## 9. Was die Planung anpassen muss (Abweichungen)

| Annahme im früheren Entwurf | PiDrive-Ist | Anpassung |
|-----------------------------|-------------|-----------|
| „Pi liefert 44.1 Stereo PCM“ | gemischte Raten; kein Capture-Contract | Resample-Node auf dem Pi erzwingen |
| „`audio route gateway` reicht“ | DAB = Direct ALSA | DAB-Arbeitspaket explizit |
| „Trigger-Dispatcher unverändert“ | ja, wenn Events über `map_event`/`pidrive_cmd` | bestätigt |
| „`pidrive_avrcp.service` parallel“ | ja für BlueZ-Fallback; bei Gateway idle | klarstellen: nicht zwei Targets zum BMW |
| „Pi kennt kein BlueZ im Gateway-Pfad“ | BlueZ-Code bleibt für Fallback/`bt` | nur Pfad-seitig wahr |

---

## 10. Empfohlene Integrationsreihenfolge (Pi-Seite)

Nach ESP Phase 0 + Coexistence + Laptop-PCM-Test:

1. PW virtual sink + `pdap_tester` (Laptop) → ESP → Kopfhörer/BMW  
2. `gateway_client` Skeleton: HELLO/AUTH/HEARTBEAT/STATUS  
3. `audio_output=gateway` + Webradio-PCM  
4. Metadata-Push  
5. AVRCP reverse via Event-Namen + `map_event`  
6. DAB-Pfad nachziehen  
7. BlueZ-BMW-Pfad nur noch Fallback

---

## 11. Top-Dateien für Implementierer

1. `pidrive/modules/audio.py`  
2. `pidrive/integration/avrcp_trigger.py` (`map_event`, CMD_FILE)  
3. `pidrive/mpris2.py`  
4. `pidrive/modules/bluetooth/bt_audio.py` / `bt_connect.py` / `bt_helpers.py`  
5. `pidrive/modules/radio/dab_play.py`  
6. `pidrive/settings.py`  
7. `pidrive/trigger/trigger_dispatcher.py` + `td_hardware.py`  
8. `pidrive/cli/cli.py`  
9. `iDriveBt.md`, `BluetoothError.md`  
10. `install.sh` / `scripts/fix-bt-a2dp.sh` (WP-Rollen: live = `a2dp_source` only)

Referenz-Clone für Planung: lokal bei Bedarf `git clone https://github.com/MPunktBPunkt/pidrive.git`.
