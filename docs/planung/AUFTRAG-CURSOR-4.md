# Anweisung 4 — Pfad-Mapping, Flash-Proxy, PiDrive-Gegenbefunde

**Stand:** 2026-09-15
**Anlass:** Review von [REVIEW-V1.3.md](REVIEW-V1.3.md) und [FLASH-BUDGET.md](FLASH-BUDGET.md).
**Vorgänger:** [AUFTRAG-CURSOR-3.md](AUFTRAG-CURSOR-3.md) — vollständig umgesetzt.
**Geprüfter PiDrive-Stand:** Repo `pidrive`, Commit `54d39ff` (M4).

Auftrag 3 ist abgearbeitet. Die Partitionstabelle stimmt arithmetisch (Offsets lückenlos,
`0x3E0000 + 0x20000 = 0x400000`), das Pflichtenheft trägt 20 × `[FIX]` und 10 ×
`[ENTWURF — Gate: …]` mit jeweils benanntem Gate. Die Korrektur von H-F4 gegen `main.js`
**v0.5.12** ist übernommen — danke, der Auftragstext lag auf v0.5.8.

Dieses Dokument liefert **drei Dinge nach**, die in Auftrag 3 fehlten oder inzwischen
überholt sind. Kapitel 1 ist die Lieferung, auf die `PIDRIVE-INTEGRATION.md:258`
ausdrücklich wartet.

---

## 1. Pfad-Mapping PiDrive — die fehlende Lieferung `[BELEGT]`

Die hochgeladene Fassung von Auftrag 3 enthielt Kapitel 7 nur als frühen Entwurf ohne
Tabelle. Deshalb steht in `PIDRIVE-INTEGRATION.md:258` korrekt „bis eine Mapping-Tabelle
geliefert wird — nicht raten". **Das Raten war richtig unterlassen. Hier ist die Tabelle.**

Der Umzug im `pidrive`-Repo ist **vollzogen** (Arbeitspakete D1–D5 durch). Im
Wurzelverzeichnis liegt nur noch `README.md`. Alle Pfade unten sind gegen den
Dateibestand von `54d39ff` verifiziert, nicht aus dem Index abgeschrieben.

| Alter Pfad (`pidrive/`) | Neuer Pfad (`pidrive/`) |
|-------------------------|-------------------------|
| `iDriveBt.md` | `docs/fahrzeug/iDriveBt.md` |
| `BluetoothError.md` | `docs/betrieb/BluetoothError.md` |
| `TROUBLESHOOTING.md` | `docs/betrieb/TROUBLESHOOTING.md` |
| `ARCHITECTURE.md` | `docs/architektur/ARCHITECTURE.md` |
| `RUNTIME_FLOWS.md` | `docs/architektur/RUNTIME_FLOWS.md` |
| `DEVELOPER_GUIDE.md` | `docs/architektur/DEVELOPER_GUIDE.md` |
| `KontextPiDrive.md` | `docs/KontextPiDrive.md` |
| `AUFTRAG-MENUE-UND-GATEWAY.md` | `docs/auftraege/AUFTRAG-MENUE-UND-GATEWAY.md` |
| `MIGRATION_BACKLOG.md` | `docs/archiv/MIGRATION_BACKLOG.md` — Archiv, nicht mehr gepflegt |
| `MIGRATION_STRUCTURE.md` | `docs/archiv/MIGRATION_STRUCTURE.md` — Archiv, nicht mehr gepflegt |

**Neu hinzugekommen** (in Auftrag 3 noch nicht existent, jetzt zitierfähig):

| Pfad | Inhalt | Relevanz hier |
|------|--------|---------------|
| `docs/README.md` | Dokumentindex inkl. Pfad-Mapping | Einstieg statt Wurzel-Verweise |
| `docs/FEATURES.md` | Funktionsvertrag (Regressionsschutz M0) | Was PiDrive zusagt |
| `docs/architektur/ZUSTANDSMASCHINE.md` | Quellen-Zustandsmaschine, Befunde Z1–Z11 | Quellenmodell für das Gateway (§3.4) |
| `docs/auftraege/AUFTRAG-WEBUI-SANIERUNG.md` | WebUI/Scanner/FastScan-Sanierung, W0–W11 | enthält C16 und die Hardware-Tests H0–H5 |
| `docs/menue/MENU-ERGONOMIE.md` | Tastendruck-Kosten des Menüs | Beleg für S3 / AVRCP-Browsing |

### 1.1 Die neun toten Verweise — abzuarbeiten

Genau diese Stellen zeigen noch auf Wurzelpfade. In **einem** Commit anpassen:

| Datei | Zeilen | Was dort steht |
|-------|--------|----------------|
| `AVRCP-MOEGLICHKEITEN.md` | 5, 110 | `pidrive/iDriveBt.md`, `pidrive/RUNTIME_FLOWS.md` |
| `PIDRIVE-INTEGRATION.md` | 43, 44, 45 | Verzeichnisbaum des PiDrive-Repos — zeigt Docs im Wurzelverzeichnis |
| `PIDRIVE-INTEGRATION.md` | 61, 157, 253 | `BluetoothError.md`, `iDriveBt.md` |
| `PIDRIVE-INTEGRATION.md` | 258 | Pfad-Hinweis — **ersetzen** durch Verweis auf dieses Kapitel |

`PHASE-0-MESSPLAN.md` §2.2 war in Auftrag 3 als betroffen genannt, ist es nach Prüfung
aber **nicht** — dort stehen keine PiDrive-Dokumentpfade. Nicht anfassen.

### 1.2 Zwei Dokumente, die es noch nicht gibt

Diese sind die Ergebnisträger der Phase −1 und werden im PiDrive-Index als geplant
geführt. Sie sind **noch nicht angelegt** — verlinke sie erst, wenn sie existieren, sonst
erzeugt ihr die nächste Generation toter Links:

| Geplanter Pfad | Inhalt | Paket |
|---|---|---|
| `pidrive/docs/fahrzeug/BMW-AVRCP-PROBE.md` | Browsing-Probe: PSM `0x001B`, SDP-Bits, PDU-Log | G1 |
| `pidrive/docs/fahrzeug/BMW-DISPLAY-PROBE.md` | Kommen die drei Metadata-Zeilen im iDrive an? | G2 |

**Namenskonvention:** `KEBAB-CASE` gilt nur für **neue** Dokumente. Bestehende Dateien
behalten ihre Namen — die zweite Umzugsstufe aus E7 findet nicht statt, das Mapping oben
ist endgültig. Die Warteschleife aus Anweisung 2 ist damit aufgehoben.

Weiterhin gilt: **im `pidrive`-Repo nichts editieren.** Der dortige Link-Prüfer
`tools/check_docs.sh` prüft keine Verweise aus diesem Repo heraus; die Liste in §1.1 ist
die einzige Absicherung.

---

## 2. Flash-Budget — Proxy schärfen, Vorbehalt ergänzen

Methode und Budgetgrenze in `FLASH-BUDGET.md` sind tragfähig. Dass die Stack-Zahlen
fehlen, ist korrekt begründet: auf dem Planungs-Host ist **keine** ESP32-Toolchain
vorhanden — geprüft wurden `idf.py`, `pio`, `platformio`, `arduino-cli`, `esptool`. Die
Weigerung, die Lücke zu schätzen, bleibt richtig.

Zwei Korrekturen an §4.2.

### 2.1 Näherer Proxy verfügbar

Als Familien-Proxy wurde `esp-hub-base.1.7.0.esp32.bin` verwendet. Im selben
Firmware-Ordner des Hub-Repos liegt ein funktional näherer Verwandter — `webradio`
bringt WiFi, HTTP/WebUI **und** eine Audio-Pipeline mit:

| Artefakt | Bytes | % von `0x1E0000` | Enthält |
|----------|------:|-----------------:|---------|
| `esp-hub-base.1.7.0.esp32.bin` | 1 201 152 | 61 % | WiFi, HTTP, WebUI |
| `webradio.2.4.0.esp32.bin` | 1 246 448 | **63 %** | + Audio-Dekodierung/-Ausgabe |
| `io-control.1.6.1.esp32.bin` | 1 349 312 | **69 %** | Familien-Maximum der Stichprobe |

Tabelle §4.2 auf **61–69 %** erweitern (Reserve 31–39 %, also ca. 610–765 KB für
Classic BT + A2DP + SBC + PDAP). `webradio` als Leitwert nennen, nicht `esp-hub-base`.

### 2.2 Der Vorbehalt, der in die andere Richtung wirkt

§4.2 liest den Proxy nur als Warnung. Es fehlt die Gegenrichtung, und die ist
entscheidungsrelevant:

**Alle diese Artefakte sind Arduino-Builds.** Ein ESP-IDF-Build gleicher Funktion fällt
typisch schlanker aus, weil das Arduino-Core-Framework unabhängig vom genutzten Umfang
mitgelinkt wird. Die 61–69 % sind für einen IDF-Gateway daher **vermutlich pessimistisch**.

Konsequenz für die Lesart: Der Proxy taugt, um das Risiko R25 zu begründen und die Spalte
„Codegröße" in A17 zu rechtfertigen. Er taugt **nicht**, um A16/Q6 zu entscheiden. Ein
8/16-MB-Modul auf Proxy-Basis zu eskalieren wäre eine Entscheidung auf der falschen
Datenlage — die Eskalationsschwelle „< 20 % Reserve" gilt für **gemessene** Werte.

Diesen Satz bitte in §4.2 „Lesart" aufnehmen.

---

## 3. PiDrive-Gegenbefunde — was die Planung berührt

Seit Auftrag 3 wurde `pidrive` tief analysiert (Audio-Pfad, Metadaten, Menü-Export,
Kommandoeingang, Zustandsmaschine). Fünf Ergebnisse widersprechen Annahmen in
`PIDRIVE-INTEGRATION.md` oder verschärfen sie. Alle gegen Commit `54d39ff` belegt.

**Kein Code in `pidrive` ändern** — das ist dort Arbeitspaket. Hier nur als Randbedingung
der Gateway-Spezifikation vermerken.

### 3.1 P-F1 Metadaten erreichen den BMW nur bei Menü-Änderung `[BELEGT]` — kritisch

`PIDRIVE-INTEGRATION.md` §6 nimmt an, PiDrive erzeuge laufend Metadaten, die das Gateway
weiterreicht. Der Auslöser ist aber falsch verdrahtet:

```
pidrive/main_core.py:712-722
    _cur_rev = menu_state.rev
    if _cur_rev != _last_menu_rev:      # Menü-Revision, nicht Titel!
        exported = menu_state.export()
        ipc.write_menu(exported)
        _last_menu_rev = _cur_rev
        if _mpris2:
            _mpris2.update(S, exported)
```

`_mpris2.update()` steht **innerhalb** der Menü-`rev`-Bedingung. Ein Titelwechsel — DAB-DLS,
Webradio-ICY, lokale Datei — ändert `menu_state.rev` nicht und löst folglich **keinen**
Metadaten-Push aus. Das Display aktualisiert erst, wenn der Nutzer im Menü navigiert.

**Folge für dieses Repo:** Die Kernfunktion „Titelanzeige im iDrive" scheitert stromabwärts,
egal wie gut das Gateway AVRCP beherrscht. Die PDAP-Metadatenrichtung braucht auf der
Pi-Seite einen **eigenen** Auslöser am Titelwechsel, nicht am Menü.

Als **P12** in `PIDRIVE-INTEGRATION.md` aufnehmen (Pi-seitiges Paket) und in §2.24 bzw. der
Metadaten-Anforderung des Pflichtenhefts als Vorbedingung markieren. Ohne P12 ist eine
Abnahme „Metadaten im Display" nicht möglich.

### 3.2 P-F2 DAB-Bypass ist unbedingt und begründet `[BELEGT]`

§4.3 führt den DAB-Pfad als offenen Punkt. Er ist härter als dort dargestellt:

```
pidrive/modules/radio/dab_play.py:363-372
    # Direktes ALSA wie manuelles `sudo welle-cli` (ohne PA-Plugin).
    # pidrive_core.service setzt PULSE_SERVER global - fuer welle-cli entfernen,
    # sonst blockiert das PipeWire-ALSA-Plugin den Decode/PCM-Pfad.
    _welle_env.pop("PULSE_SERVER", None)
    _welle_env.pop("PULSE_SINK", None)
```

Zusätzlich schreibt das Modul eine `asound.conf`, die den ALSA-Default auf die Klinke
zwingt (`dab_play.py:383`). Der Bypass ist also **nicht** Bequemlichkeit, sondern Umgehung
eines ungelösten PipeWire-Fehlers, und er ist **bedingungslos** — es gibt keinen Schalter.

**Folge:** Ein Gateway-Ausgang, der Audio über PipeWire abgreift, bekommt von DAB **nichts**.
Das Gate `[ENTWURF — Gate: A7 + PipeWire-Bridge auf dem Pi]` in Pflichtenheft §L435 ist
damit nicht bloß ein Integrationsschritt, sondern setzt die Behebung eines bestehenden
Fehlers voraus. Formulierung dort verschärfen: Gate ist **A7 + Behebung des
PipeWire-Blockade-Fehlers**, nicht nur „Bridge vorhanden".

### 3.3 P-F3 Der Pi ist reine A2DP-Source `[BELEGT]`

Für Mehrquellen-Betrieb (Handy, Tablet → Pi) müsste der Pi A2DP-**Sink** sein. Die
Installation schließt das aus:

```
pidrive/install.sh:687-692
    # Nur a2dp_source - kein hfp_ag/Telephony
    bluez5.roles = [ a2dp_source ]
    bluez5.auto-connect = [ a2dp_source ]
```

Das war in Auftrag 3 die Grundlage für die Empfehlung, die Sink-Rolle auf den Pi zu
verlagern, nachdem der BMW am ESP hängt. Diese Empfehlung bleibt richtig — aber sie ist
**kein Nullaufwand**: die WirePlumber-Rollen müssen erweitert werden, und die Konfiguration
wird im Installer **inline** erzeugt (die Datei in `pidrive/pipewire-config/` ist toter
Code). Das gehört als Aufwandsvermerk an A-Position „Mehrquellen" statt als erledigt.

### 3.4 P-F4 `bt` existiert im Quellenmodell nicht `[BELEGT]`

`pidrive/docs/architektur/ZUSTANDSMASCHINE.md`, Befund Z1: Bluetooth kommt in **keinem**
`commit_source()`-Aufruf vor. Der Zustand lebt in Parallelfeldern (`bt_state`,
`bt_link_state`, `bt_audio_state`) außerhalb der Quellen-Zustandsmaschine.

**Folge:** Das Gateway wird ebenfalls eine Quelle. Wenn es nach dem Vorbild von `bt`
modelliert wird, erbt es dieselbe Lücke. Die Gateway-Quelle muss über `commit_source()`
laufen. Die Vorarbeit ist Stufe 2 in `ZUSTANDSMASCHINE.md` — als Vorbedingung in
`PIDRIVE-INTEGRATION.md` verlinken.

### 3.5 P-F5 Menübaum 98 KB ohne Paginierung `[BELEGT]`

`pidrive/tests/golden/menu_tree.json` = **97 915 B**. Der Export bietet keine Seitenbildung:
`menu_state.py:322 export()` und `:342 export_tree()` nehmen keine `offset`/`limit`/`page`-Parameter.

**Folge für den PDAP-Menükanal:** Ein vollständiger Baumtransfer ist auf dem ESP32 nicht
haltbar — knapp 100 KB JSON überschreiten den komfortabel verfügbaren Heap, und AVRCP
`GetFolderItems` fragt ohnehin seitenweise (`start_item`/`end_item`) ab. Der PDAP-Menükanal
braucht daher **zwingend** ein seitenbasiertes Abrufmodell (`uid` + `offset` + `count`),
kein „Baum einmal laden".

Die PDAP-Skizze in Auftrag 3 ist an dieser Stelle zu konkretisieren; die Pi-Seite braucht
eine paginierte Exportfunktion (Paket in `pidrive`). Bitte als Anforderung im
Pflichtenheft-Kapitel zum Menükanal verankern und in den Nicht-Zielen festhalten:
**kein vollständiger Baumtransfer über PDAP.**

### 3.6 P-F6 Der Rückmeldekanal für Aktionen endet vor dem Fahrzeug `[BELEGT]`

**Herkunft:** Der Eigentümer bemerkte am 2026-09-15, dass die Menüeinträge „System-Info"
und „Version" am BMW nichts bewirken. Die Prüfung ergab eine Lücke, die weit über diese
zwei Einträge hinausgeht.

PiDrive meldet Fortschritt und Ergebnisse von Aktionen über `ipc.write_progress()`
(`pidrive/ipc.py:131`) nach `/tmp/pidrive_progress.json` (`ipc.py:15`). Diese Datei lesen
`cli/adapters.py`, `cli/cli.py`, `cli/service.py` und `web/shared/constants.py` — also
**nur CLI und WebUI**. Gegenprobe: weder `mpris2.py` noch `main_core.py` enthalten das
Wort `progress`.

Betroffen sind **104 Aufrufstellen**, darunter `bt_connect.py` (19), `td_radio.py` (16),
`td_hardware.py` (15), `update.py` (12), `audio.py` (9), `scanner.py` (8), `wifi.py` (7).
Im Fahrzeug ist damit weder „Gerät verbunden" noch „Suchlauf 7/38" noch „RTL-SDR belegt"
sichtbar.

**Wichtige Abgrenzung zu P-F1:** ein Metadaten-Push findet sehr wohl statt.
`check_trigger()` (`main_core.py:177-209`) meldet jedes behandelte Kommando als
verarbeitet, `rebuild_tree()` läuft und erhöht den Revisionszähler
(`menu_state.py:290`). Der Push trägt nur keine neue Information, weil die Nutzlast im
Overlay steckt. Für das Gateway heißt das: der Kanal ist vorhanden und wird bedient — es
fehlt ein **Nachrichtentyp**, nicht ein Auslöser.

**Folge für das Gateway.** Über die drei AVRCP-Metadatenfelder ist ein Fortschrittsbalken
nicht abbildbar — das ist der Grund, warum die Lücke bisher niemand geschlossen hat. Der
PDAP-Menükanal steht damit vor einer Anforderung, die in der bisherigen Planung fehlt:
**neben Menüknoten und Wiedergabemetadaten braucht er einen dritten Nachrichtentyp für
transiente Meldungen** (Hinweis, Fehler, Fortschritt) mit Gültigkeitsdauer.

Konkret in die PDAP-Skizze aufzunehmen:

| Feld | Zweck |
|---|---|
| `kind` | `info` \| `warn` \| `error` \| `progress` — entspricht dem `color`-Parameter von `write_progress()` |
| `title`, `text` | die beiden vorhandenen Textfelder |
| `pct` | optional, nur bei `kind=progress` — PiDrive liefert es schon |
| `ttl_ms` | Anzeigedauer; PiDrive räumt heute nach 3–4 s mit `clear_progress()` auf |

Damit kann das Gateway selbst entscheiden, wie es die Meldung darstellt — als
AVRCP-Titelzeile bei S1, als eigenes Listenelement bei S3. Diese Entscheidung ist **erst
nach Phase −1** zu treffen; die Datenstruktur ist davon unabhängig und sollte jetzt
festgelegt werden, weil sie den Menükanal-Vertrag betrifft.

Auf der Pi-Seite ist der Befund als **D1** in
`pidrive/docs/auftraege/AUFTRAG-DISPLAY-RUECKMELDUNG.md` erfasst, die Gestaltungsfrage als
dortige Entscheidung **E-D1**. Bitte **keine** Änderung in `pidrive` vornehmen (siehe §4);
der Verweis genügt.

### 3.7 P-F7 Der Kommandokanal hat einen nicht angemeldeten Mitleser `[BELEGT]`

**Für den PDAP-Kommandokanal die härtere Randbedingung von beiden.** Gateway-Kommandos
landen auf der Pi-Seite in `/tmp/pidrive_cmd`. Diese Datei hat aber nicht einen Leser,
sondern zwei konkurrierende — und der zweite ist nicht angemeldet.

Drei Eingabeschleifen lesen `ipc.CMD_FILE` direkt und löschen sie anschließend, statt
`ipc.drain_triggers()` zu benutzen:

| Ort | Funktion | Dauer |
|---|---|---|
| `fm.py:370-375` | `freq_input_screen()` | bis 60 s |
| `scanner.py:718-723` | `_freq_input()` | modal |
| `scanner.py:761-767` | `freq_input_screen()` | modal |

Für modale Zustände existiert der `LIST_FILE`-Mechanismus: `main_core.py:181-193` wertet
ihn aus und gibt Navigationskommandos bewusst an die modale Schleife ab.
`local_player.py:47` benutzt ihn korrekt. Die drei Frequenzeingaben tun es **nicht** — sie
greifen ohne Sperre auf dieselbe Datei zu wie `main_core.py:188-190`. Wer ein Kommando
bekommt, ist nichtdeterministisch.

**Folge für das Gateway.** Solange diese Schleifen bestehen, kann ein über PDAP gesendetes
Kommando verschwinden, ohne dass es eine Fehlermeldung gibt — und zwar mit
Wahrscheinlichkeit, nicht deterministisch. Ein Quittungsmechanismus im Protokoll
(„Kommando angenommen") hilft dabei nur begrenzt: der Verlust passiert **nach** dem
Schreiben der Datei.

Daraus folgen zwei Anforderungen:

1. **Der PDAP-Kommandokanal braucht eine Quittung mit Bezug auf das Kommando**, nicht nur
   auf den Transport — also eine Rückmeldung „`goto:uid` ausgeführt, neue Revision N"
   statt eines TCP-ACK. Nur damit kann das Gateway einen Verlust überhaupt erkennen.
2. **Der modale Zustand muss über PDAP sichtbar sein.** Ist auf der Pi eine
   Eingabeschleife aktiv, muss das Gateway es wissen — sonst schickt es Kommandos in einen
   Zustand, der sie anders interpretiert. Das lässt sich an den Meldungstyp aus §3.6
   anhängen (`kind=modal` mit Restlaufzeit).

Auf der Pi-Seite ist der Befund als **D3** in
`pidrive/docs/auftraege/AUFTRAG-DISPLAY-RUECKMELDUNG.md` erfasst; die Korrektur dort (Anmeldung über
`LIST_FILE` oder Wegfall der Menü-Frequenzeingabe) ist **Vorbedingung** für einen
verlässlichen PDAP-Kommandokanal. Bitte in `PIDRIVE-INTEGRATION.md` als solche verlinken.

Ergänzend gilt **P-F8** aus derselben Prüfung (dort **D2**): Radioaktionen stoßen nach
Abschluss keinen zweiten Push an, weil `fm.py`, `dab_play.py`, `scanner.py`, `webradio.py`
und `favorites.py` `S["menu_rev"]` **null** mal erhöhen — `bt_connect.py` dagegen zwölf
mal. Für das Gateway heißt das: **nach einem Kommando nicht auf einen Push warten.** Der
Menükanal braucht ein aktives Abfragen der Revision, keine reine Benachrichtigungslogik.

---

## 4. Was ausdrücklich nicht zu tun ist

- **Keine Firmware**, kein `main/`, keine `components/`, keine `partitions.csv` vor A17.
- **Keine Änderung** in `pidrive` oder `iobroker.esp-hub` — auch nicht an Doku.
- **Keine A17-Entscheidung** ohne Phase −1 **und** gemessene Codegröße. Der Proxy aus §2
  ersetzt die Messung nicht.
- **Keine Eskalation auf 8/16 MB** auf Proxy-Basis (§2.2).
- `0x1E0000` **nicht** vergrößern, `logs` nicht in App-Slots umwidmen.
- Die beiden Dokumente aus §1.2 **nicht** verlinken, solange sie nicht existieren.
- Keine neue Freigabe-Checkliste neben Abschnitt E.

---

## 5. Owner-Fragen

Q1–Q5 (Anweisung 2) und Q6–Q11 (Anweisung 3) bleiben offen. Neu:

| # | Frage | Warum sie blockiert |
|---|-------|---------------------|
| **Q12** | **Wo werden die ESP32-Projekte gebaut?** Auf dem Planungs-Host ist keine Toolchain installiert. Existiert eine Maschine mit ESP-IDF oder PlatformIO, auf der die vier Builds aus `FLASH-BUDGET.md` §3 laufen können? | Ohne echte Zahlen bleibt **A17** blockiert, und A17 blockiert jede Firmware. Das ist derzeit der längste Pfad im Projekt. |
| **Q13** | Soll der Gateway-Menükanal von Anfang an paginiert spezifiziert werden (§3.5), oder erst nach dem Phase-−1-Ergebnis? | Bei Browsing-Erfolg (S3) diktiert AVRCP das Seitenmodell; bei S1 wäre eine einfachere Form vertretbar. |
| **Q14** | Welche Meldungen sollen es über PDAP bis zum Fahrer schaffen (§3.6)? | Empfehlung: **Fehler und Warnungen ja, Fortschritt nein.** „Gerät nicht verbunden" und „RTL-SDR belegt" lassen den Fahrer heute ratlos; „Suchlauf 7/38" ist über drei Textfelder ohnehin nur als Zahl darstellbar und eher Ablenkung. Die Datenstruktur deckt beides ab, die Auswahl trifft PiDrive. |

Zu Q12: Der Fahrzeug-Pi (`192.168.178.105`) ist als Build-Host **nicht** empfohlen — er ist
ARM64 und ESP-IDF läuft dort grundsätzlich, aber eine mehrere Gigabyte große Toolchain auf
dem produktiven Autorechner zu installieren schafft ein Problem, um ein anderes zu lösen.

**Q12 beantwortet (2026-09-15):** Die zweite Cursor-Instanz verfügt über eine Toolchain
(ESP-IDF oder PlatformIO). Damit sind die vier Vergleichsbauten aus
[FLASH-BUDGET.md](FLASH-BUDGET.md) §3 durchführbar — die Flash-Messung ist nicht mehr
blockiert.

**A17** hängt damit nur noch an zwei Dingen: der Flash-Messung und dem Ergebnis von
Phase −1. Für Phase −1 liegt das Werkzeug inzwischen im PiDrive-Repo
(`tools/bmw_avrcp_probe.sh` + `tools/bmw_avrcp_analyze.py`, Paket G1). Der
Interpretationsschlüssel ist im Auswerteskript **vorab** festgeschrieben, damit das Urteil
nicht davon abhängt, was man sich beim Lesen der Rohdaten wünscht.

Daraus folgt die Reihenfolge: Flash-Messung und Fahrzeugmessung laufen **parallel**, beide
münden in A17. Vorher entsteht keine Firmware.

---

## 6. Reihenfolge

1. **Kapitel 1** — Pfad-Mapping übernehmen, die neun Verweise in einem Commit korrigieren,
   `PIDRIVE-INTEGRATION.md:258` ersetzen. Sofort machbar, keine Abhängigkeit.
2. **Kapitel 2** — `FLASH-BUDGET.md` §4.2 um `webradio`-Proxy und Arduino-Vorbehalt ergänzen.
3. **Kapitel 3** — P-F1 bis P-F5 als P12 / Gate-Verschärfung / Nicht-Ziel verankern.
   P-F1 und P-F5 sind die wirksamen; P-F1 gehört als Vorbedingung in die
   Metadaten-Abnahme, P-F5 in den Menükanal.
4. **Q12 klären** — parallel, durch den Owner.
5. Danach unverändert: Flash-Messung → Phase −1 → A17.

---

## 7. Historie

| Version | Datum | Inhalt |
|---------|-------|--------|
| 4.0 | 2026-09-15 | Pfad-Mapping nachgeliefert; Flash-Proxy geschärft (`webradio`, Arduino-Vorbehalt); PiDrive-Gegenbefunde P-F1–P-F5; Q12/Q13 |
