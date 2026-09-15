# Auftrag & Planungsbeitrag: BT-Steuerung des iDrive, Stack-Wahl, Menü-Transport

**Adressat:** zweite Cursor-Instanz, die im Repo `esp32.bt-gateway` arbeitet
**Stand der Analyse:** Planung V1.1 · `pidrive` v0.11.127
**Erstellt:** 2026-09-15
**Gegenstück:** `pidrive/AUFTRAG-MENUE-UND-GATEWAY.md`

---

## 0. Rolle dieses Dokuments

Wir sind in der **Ideenfindung**, nicht in der Umsetzung. Dieses Dokument tut zwei Dinge:

1. Es bringt **verifizierte technische Befunde** in die Planung ein, die an mehreren Stellen
   Annahmen von V1.1 widerlegen oder erweitern.
2. Es gibt der zweiten Instanz **konkrete Redaktionsaufträge** für die Planungsdokumente.

**Es wird keine Firmware begonnen.** Alle Aufträge sind Planungs- und Messaufträge. Wer
anfängt, ESP-IDF-Code zu schreiben, arbeitet gegen den Auftrag — die Stack-Entscheidung
(A17 unten) ist offen, und sie entscheidet über die gesamte Komponentenstruktur.

---

## 1. Was an V1.1 tragfähig ist

Kurz, damit es beim Umbau nicht verloren geht: Die Verantwortungsgrenzen (§2.1), das
Coexistence-Gate als hartes Kriterium (§2.4), „Observe first", die Ein-Session-Regel, die
PiDrive-Ist-Analyse inklusive der DAB-Direct-ALSA-Falle und die Entscheidung, den ESP über
`iobroker.esp-hub` programmierbar zu halten — das alles bleibt. Die folgenden Befunde
greifen die **Bluetooth-Schicht** und das **Menü** an, nicht das Grundkonzept.

---

## 2. Verifizierte Befunde, die die Planung ändern

### F1 — Bluedroid kann als AVRCP Target keine Metadaten senden

Pflichtenheft §2.10 setzt voraus: „ESP setzt daraus AVRCP TrackChanged + Element
Attributes". Das ist mit ESP-IDF/Bluedroid nicht umsetzbar.

Die öffentliche Target-API (`esp_avrc_api.h`) kennt genau sieben Events —
`CONNECTION_STATE`, `REMOTE_FEATURES`, `PASSTHROUGH_CMD`, `SET_ABSOLUTE_VOLUME_CMD`,
`REGISTER_NOTIFICATION`, `SET_PLAYER_APP_VALUE`, `PROF_STATE` — und keine Funktion, um
Titel/Interpret/Album zu setzen. `GetElementAttributes` (PDU 0x20) wird nicht an die
Anwendung gereicht. `esp_avrc_md_attr_mask_t` und `esp_avrc_ct_send_metadata_cmd()`
existieren nur für die **Controller**-Rolle, also für den Fall „ESP fragt ein Handy".
`esp_avrc_tg_send_rn_rsp()` kann zwar `TRACK_CHANGE` melden, aber die Attributwerte, die
das BMW danach anfordert, kann die Anwendung nicht liefern.

Der einzige bekannte Weg ist ein Patch in
`components/bt/host/bluedroid/bta/av/bta_av_act.c` (`bta_av_proc_meta_cmd()`), also ein
Eingriff in den IDF-Baum mit eigener Attribut-Kodierung (Charset 0x006A).

**Konsequenz, die in der Planung fehlt:** Im Gateway-Pfad wäre das BMW-Display **leer** —
und damit fällt genau das Feature weg, das PiDrive gerade erst reparieren konnte (der
`art_url`-Bug in `mpris2.py`, gefixt in v0.11.123). Das ist kein Komfortverlust, sondern der
Verlust der Bedienoberfläche: Das PiDrive-Menü *ist* die Metadatenanzeige.

### F2 — Bluedroid hat keinen AVRCP-Browsing-Kanal

`ESP_AVRC_FEAT_BROWSE` existiert als Feature-Bit, aber es gibt keine Browsing-Kommandos
oder -Callbacks in der API. Espressif führt das als offenes Feature (Issue IDFGH-6757,
espressif/esp-idf#8385) und nennt dort ausdrücklich nur die **Controller**-Rolle als Ziel;
Browsing braucht zudem L2CAP ERTM.

**Konsequenz:** Ein echtes Listen-Menü im iDrive ist mit Bluedroid ausgeschlossen —
unabhängig davon, ob das Fahrzeug es könnte.

### F3 — BTstack kann beides

BTstack (BlueKitchen) hat auf der Target-Seite:

- `avrcp_target_set_now_playing_info()` + `avrcp_target_track_changed()` → Metadaten
  aufs Display,
- `avrcp_browsing_target_*` mit `SetBrowsedPlayer`, `ChangePath`, `GetFolderItems`,
  `GetItemAttributes`, `Search`, `PlayItem`, `GetTotalNumberOfItems` → echtes Listen-Menü.

Weitere relevante Fakten: aktiv gepflegter ESP32-Port (`port/esp32`, getestet mit
ESP-IDF 5.4, eigener Buildbot), A2DP-Source-Beispiel vorhanden, SIG-Qualifikation u. a. für
A2DP 1.4, AVCTP 1.4, **AVRCP 1.6.3**. Lizenz: frei für nicht-kommerzielle Nutzung,
kommerziell lizenzpflichtig — für ein GPL-3.0-Hobbyprojekt passend, muss aber bewusst
festgehalten werden.

Einschränkung, die zu prüfen ist: Der ESP32-Port nutzt den mitgelieferten Controller über
VHCI; im Port-README sind bekannte Controller-Probleme vermerkt. Das betrifft das
Coexistence-Gate (§2.4) und ist ein eigener Prüfpunkt, kein Ausschlussgrund.

### F4 — A2DP Sink und Source gleichzeitig geht auf dem ESP32 nicht

Espressif dokumentiert das in **beiden** A2DP-Beispielen wörtlich: „A2DP source cannot be
used together with A2DP sink at the same time". Damit ist der Wunsch des Eigentümers,
„mehrere Audioquellen (Handy, evtl. Tablet) am Gateway zu koppeln", auf dem ESP nicht
erfüllbar — jedenfalls nicht mit Bluedroid, und mit BTstack nur unter erheblichem Risiko
(zwei ACL-Links, zwei A2DP-Streams, SBC-Dekodierung *und* -Kodierung, plus WiFi, auf einem
WROOM).

**Der Wunsch ist trotzdem erfüllbar** — über den Pi: Sobald das BMW am ESP hängt, wird der
CSR-Dongle des Pi frei. Handy und Tablet koppeln sich mit dem **Pi** (A2DP Sink), PipeWire
führt das Audio in denselben Capture-Punkt wie DAB und Webradio, und die Quellenumschaltung
macht das bestehende PiDrive-Menü. Architektonisch ist das nur eine neue Quelle neben
DAB/FM/Webradio — kein neues Konzept, keine neue Rolle im ESP, und der Eigentümer hat
ausdrücklich zugestimmt, dass das der richtige Weg ist. Das Arbeitspaket liegt im
`pidrive`-Repo (dort G3).

### F5 — Der Pi kann die Browsing-Frage messen, aber kein Browsing-Menü liefern

BlueZ bewirbt auf der Target-Seite den Browsing-Kanal im SDP-Record
(`avrcp_tg_record(browsing)` setzt `AVRCP_FEATURE_BROWSING` und den Additional Protocol
Descriptor mit AVCTP-Browsing-PSM), beantwortet aber nur `GetFolderItems` und
`GetTotalNumberOfItems` und dort **ausschließlich Scope 0x00 (Media Player List)**. Die
Scopes 0x01 (Virtual Filesystem), 0x02 (Search) und 0x03 (Now Playing) werden mit
`AVRCP_STATUS_INVALID_SCOPE` abgelehnt. `SetBrowsedPlayer`, `ChangePath`,
`GetItemAttributes` und `PlayItem` haben target-seitig gar keinen Handler.
(Quelle: `bluez/profiles/audio/avrcp.c`, Tabelle `browsing_handlers`.)

**Das ist die gute Nachricht:** Weil BlueZ Browsing *bewirbt*, wird das BMW es versuchen,
falls es das kann — und genau das ist messbar, mit vorhandener Hardware, ohne eine Zeile
Firmware. Der Punkt, an dem BlueZ aufgibt, ist exakt die Lücke, die ein eigener
AVRCP-Target-Stack füllen würde.

---

## 3. Das eigentliche Thema: was die BT-Steuerung des iDrive hergibt

Zusammenfassung des Ist-Zustands aus `pidrive/RUNTIME_FLOWS.md` Abschnitt I und
`pidrive/iDriveBt.md` §3:

Die iDrive-Bedieneinheit (`MENU`, `BACK`, `OPTION`, Dreh-Drück-Regler) steuert die
**BMW-eigene** Oberfläche und wird nicht als AVRCP weitergereicht. Zuverlässig kommen nur
die Medien-Transportkommandos an, praktisch also Skip vor/zurück und Play/Pause. Sichtbar
sind drei Textzeilen (Titel/Interpret/Album), und zwar immer nur der **markierte** Eintrag,
nie die Liste. Daraus ist die heutige Bedienung entstanden: Skip = Cursor, Play = Enter,
ein expliziter „Zurueck"-Eintrag in jedem Ordner als Ersatz für die fehlende Stop-Taste,
Bestätigungsebenen gegen Fehlauslösung, Doppeltipp als Sprung an den Menüanfang.

Damit sind drei Ausbaustufen denkbar:

| Stufe | Transport | Was der Fahrer sieht | Voraussetzung |
|---|---|---|---|
| **S1** heute | AVRCP Metadata (3 Zeilen) | markierter Eintrag, Skip/Play | funktioniert prinzipiell; Target-Metadaten nötig → F1 |
| **S2** | S1 + Player Application Settings | zusätzliche Auswahlwerte, die das BMW als eigene Einstellungen rendert | BMW-abhängig, schmal, eher Kuriosität |
| **S3** Ziel | AVRCP Browsing (PSM 0x001B) | **echte Liste**, vom iDrive gerendert, mit Dreh-Drück-Regler bedienbar | BMW muss Browsing als Controller beherrschen → Messung; Target-Stack mit Browsing → F2/F3 |

Bei S3 verschwinden nebenbei fast alle Krücken: Der „Zurueck"-Eintrag wird unnötig (das
Auto navigiert selbst zurück), die Info-Knoten-Blindstellen entfallen, lange Senderlisten
werden scrollbar statt durchklickbar, und die Frage „wie viele Tastendrücke bis ROCK FM"
stellt sich nicht mehr. **S3 ist der Grund, das Thema überhaupt zu verfolgen** — und die
Machbarkeit hängt an einer einzigen, billigen Messung.

Was auch bei S3 **nicht** geht, damit keine falschen Erwartungen entstehen: Cover-Art
braucht AVRCP 1.6 mit BIP/OBEX (PiDrives eigener Test meldet das seit v0.11.126 korrekt als
„erst ab 1.6"), Freitexteingabe gibt es nicht, und Aktionen ohne Wiedergabe (Reboot,
WiFi-Scan) müssen als „Medienelement, das etwas tut" modelliert werden — das Auto erwartet
nach `PlayItem` Wiedergabe. Wie man Aktionen und Schalter in einer Medienliste sauber
abbildet, ist eine offene Designfrage (A18 unten).

---

## 4. Redaktionsaufträge an die Planungsdokumente

Reihenfolge einhalten; jede Änderung als eigener Commit mit Bezug auf den Befund (F1–F5).

### 4.1 Neue Datei: `AVRCP-MOEGLICHKEITEN.md`

Das ist das Dokument, das in V1.1 fehlt — die Analyse der BT-Steuerung des iDrive als
eigenständige Grundlage, nicht verstreut in Pflichtenheft und PiDrive-Integration.

Inhalt: Rollenbild (BMW = AVRCP Controller + A2DP Sink, Gateway = Target + Source), die
drei Kanäle (AVCTP Control, AVCTP Browsing, Cover-Art/OBEX) mit jeweils „was liefert er
für die Bedienung", die Ausbaustufen S1–S3 aus Kapitel 3 mit Voraussetzungen, die
Stack-Matrix aus 4.4, und am Ende die offenen Messfragen mit Verweis auf den Messplan.
Tabellen aus `pidrive/iDriveBt.md` §2/§3 referenzieren, nicht kopieren — die Datei dort ist
die gepflegte Wahrheit über das Fahrzeug.

### 4.2 Neue Datei: `PHASE-0-MESSPLAN.md`

Phase 0 ist in V1.1 als ESP-Analyzer beschrieben (§2.3). Das ist zu spät und zu teuer: Die
architekturentscheidende Messung geht heute mit dem Pi. Neue Struktur:

| Phase | Wer | Was | Blockiert |
|---|---|---|---|
| **Phase −1** (neu) | Pi, BlueZ, `btmon` | Öffnet das NBT Evo den Browsing-Kanal? Sendet es `SetBrowsedPlayer`/`GetFolderItems`? Welche SDP-Feature-Bits (59 Browsing, 60 Searching, 65 NowPlaying)? Kommen die drei Textzeilen im Fahrzeug an? Welcher AVRCP-Eingang wird bedient? | A17, A18, S3 |
| **Phase 0** | ESP | Pass-Through-PDUs, Timing, Codec-Negotiation, Disconnect-Reasons über Zündzyklen | Mapping-Tabelle |
| **Coexistence-Gate** | ESP | wie §2.4, plus BTstack-Controller-Eigenheiten | alles Weitere |

Der Durchführungsteil von Phase −1 liegt im `pidrive`-Repo (dort G1/G2, inklusive
`tools/bmw_avrcp_probe.sh` und dem vorab festgeschriebenen Interpretationsschlüssel).
Hier nur verlinken und die **Entscheidungswirkung** dokumentieren:

- Kein Browsing-Kanal → S3 gestrichen, Zielbild bleibt S1, Stack-Frage entspannt sich
  (Bluedroid + Metadaten-Patch wird vertretbar).
- Browsing-Kanal mit `SetBrowsedPlayer` → S3 wird Zielbild, BTstack wird zur
  Standardoption, PDAP braucht einen Menü-Kanal (A18).

### 4.3 `PFLICHTENHEFT.md` — Änderungen

| Abschnitt | Änderung |
|---|---|
| §2.2 Nicht-Ziele | „A2DP Sink + Source gleichzeitig (kein Handy-BT-Relay)" präzisieren zu: **„Relay nicht im ESP** — Multi-Source (Handy, Tablet) läuft über den Pi als A2DP-Sink". Begründung F4 mit Espressif-Zitat. Das ist keine Aufweichung: der ESP bleibt Ein-Rollen-Gerät, der Wunsch wird anderswo erfüllt. |
| §2.9 AVRCP | Browsing als **Ausbaustufe S3** aufnehmen, inkl. der Target-PDUs, die dann zu implementieren sind. Heute steht dort nur Pass-Through-Mapping. |
| §2.10 Metadata-Engine | Voranstellen, dass die Umsetzbarkeit **stackabhängig** ist (F1), mit Verweis auf A17. Ohne diesen Satz ist der Abschnitt irreführend. |
| §2.3 Phase 0 | Auf `PHASE-0-MESSPLAN.md` verweisen, Phase −1 davorstellen. |
| §2.8 PDAP | Menü-/Item-Kanal als optionalen Kanal aufnehmen (Skizze in 4.6). |
| **neu §2.24** | „Menü-Transport" — eigener Abschnitt, weil das Menü das eigentliche Produktmerkmal ist und heute in §2.10 versteckt liegt. |
| §2.20 Exit-Kriterien | Zeile für „Menü im Fahrzeug bedienbar" ergänzen, mit der Ergonomie-Kennzahl aus dem PiDrive-Repo (Tastendrücke bis Ziel) als messbarem Kriterium. |
| §2.16 / Ordnerstruktur | Komponentenliste erst nach A17 festschreiben. Bei BTstack heißen die Komponenten anders (kein `a2dp_source`/`avrcp_target` als Eigenbau, sondern BTstack-Profile + eigener `menu_target`). |

### 4.4 `OFFENE-PUNKTE.md` — neue Entscheidungen und Risiken

**A1 neu bewerten.** Die Empfehlung „ESP-IDF + Bluedroid" ist mit F1/F2 nicht mehr
haltbar, solange Metadaten und Menü Ziel sind. A1 bleibt (ESP-IDF als Build-System und
RTOS), aber die Host-Stack-Frage wird aus A1 herausgelöst:

**Neu A17 — Bluetooth-Host-Stack.** Entscheidungsvorlage mit drei Optionen:

| Option | Metadaten | Browsing | Aufwand | Risiko |
|---|---|---|---|---|
| Bluedroid unverändert | ⛔ | ⛔ | gering | Display bleibt leer → Projektziel verfehlt |
| Bluedroid + Patch in `bta_av_act.c` | ⚠ per Patch | ⛔ | mittel | Fork-Pflege bei jedem IDF-Update; S3 dauerhaft verbaut |
| **BTstack auf ESP32** | ✅ API | ✅ API | höher (neuer Stack, eigener Coexistence-Nachweis) | Controller-Eigenheiten des ESP32-Ports; Lizenz nicht-kommerziell |

Empfehlung: **BTstack**, sobald Phase −1 zeigt, dass Metadaten im Fahrzeug ankommen — und
zwingend, falls Browsing möglich ist. Die Entscheidung ist erst nach Phase −1 zu treffen,
aber die Vorlage muss jetzt stehen.

**Neu A18 — Menü-Transport und Semantik.** Wenn S3 kommt: Wie werden Ordner, Sender,
Aktionen und Schalter auf Folder-Items und Media-Elements abgebildet? Wie werden Aktionen
modelliert, bei denen das Auto nach `PlayItem` Wiedergabe erwartet? Wird der Baum
vollständig übertragen oder seitenweise nachgeladen (`GetFolderItems` ist ohnehin
paginiert)? Wie wird `uid_counter` invalidiert, ohne dass das Auto bei jedem BT-Statuswechsel
die Liste neu zieht? Vorschlag: Baum vollständig, aber mit Größenbudget (siehe 4.6).

**Neu A19 — Multi-Source über den Pi.** Formal festhalten, dass Handy/Tablet über den
Pi-Sink laufen (F4), inklusive der Folge, dass PiDrive dann AVRCP **Controller** gegenüber
dem Handy wird und Skip-Kommandos vom BMW durchreichen muss. Das ist ein neuer Fall in
PiDrives `map_event()` und gehört in den Event-Contract (§2.9/§8 der PiDrive-Integration).

**Neu A20 — Wer ist AVRCP-Anzeigename und Player?** Bei S3 taucht der Gateway in der
Player-Liste des Autos auf. Zusammen mit A15 (BT-Anzeigename) entscheiden, wie das im
iDrive heißt und ob während des Parallelbetriebs zwei Player sichtbar sind.

**Neue Risiken:**

| # | Risiko | Früher Nachweis |
|---|---|---|
| R18 | Bluedroid-TG liefert keine Metadaten → BMW-Display bleibt leer | F1 verifiziert; Entscheidung A17 |
| R19 | NBT Evo öffnet keinen Browsing-Kanal → S3 entfällt | Phase −1 |
| R20 | Sink+Source auf einem ESP32 unmöglich → Multi-Source-Ziel nur über Pi | F4 verifiziert; A19 |
| R21 | BTstack-ESP32-Port: Controller-Eigenheiten, Coexistence unbekannt | Gate-Test §2.4 mit BTstack wiederholen, nicht von Bluedroid-Ergebnissen ableiten |
| R22 | BTstack-Lizenz (nicht-kommerziell) kollidiert mit späterer Weitergabe | vor A17 entscheiden und in `LICENSE`/README vermerken |
| R23 | Menübaum sprengt den RAM des WROOM bei großen Listen (lokale Musik) | Größenbudget in A18, Lazy-Loading als Rückfalloption |

### 4.5 `BETRIEBSMODI.md` und `KONZEPT.md`

- `BETRIEBSMODI.md` §2.1: Weg **E** („A2DP-Relay") von „nicht V1" auf **„nicht im ESP,
  dauerhaft"** setzen und einen neuen **Weg F** ergänzen: „Handy/Tablet → Pi als A2DP-Sink
  → PDAP → ESP → BMW", mit Kennzeichnung als **Hauptweg für Fremdgeräte**. Damit wird aus
  einem Nicht-Ziel ein geplanter Weg, ohne den ESP zu belasten.
- `KONZEPT.md` §1.5, Designregel 1 („Der ESP weiß nichts über DAB, Spotify, Sender,
  Menüs …"): Bei S3 muss der ESP eine **generische, semantikfreie Item-Liste**
  durchreichen. Die Regel wird zu: „Der ESP kennt keine PiDrive-Geschäftslogik; er
  transportiert eine generische Baumstruktur, deren Bedeutung ausschließlich der Client
  kennt." Das erhält die Trennung und macht S3 möglich.
- `KONZEPT.md` §1.2/§1.3: Die Nutzenliste um den Punkt ergänzen, der bisher fehlt — **das
  Gateway ist nicht nur Stabilisierung, es ist die einzige Chance auf ein echtes
  Bedienmenü im Fahrzeug.** Das ist das stärkste Argument für das ganze Projekt und steht
  heute nirgends.
- `STATE.md` und `README.md`: Status auf „Planung V1.2 (BT-Steuerung analysiert, Stack
  offen, Phase −1 ausstehend)" ziehen.

### 4.6 PDAP-Skizze für den Menü-Kanal (Diskussionsgrundlage, nicht festschreiben)

Fachlich anschlussfähig an PiDrives kommende UID-Struktur (dort M1):

```
MENU_TREE      { uid_counter, nodes: [ { uid, parent_uid, kind: folder|item,
                                         name, playable, attrs: {…} } ] }
MENU_INVALIDATE{ uid_counter }                      Pi → ESP: Baum neu holen
MENU_ACTIVATE  { uid }                              ESP → Pi: aus PlayItem
MENU_PAGE_REQ  { parent_uid, start, count }         optional, falls Lazy-Loading
MENU_PAGE_RSP  { parent_uid, start, nodes[] }
```

Zwei Punkte, die früh geklärt werden müssen:

- **Größenbudget.** Der heutige Baum hat grob 100–200 Knoten (Wurzel, Quellen, plus
  Senderlisten; DAB flach). Bei ~80 Byte pro Knoten sind das 8–16 KB — auf einem WROOM
  machbar, aber lokale Musikbibliotheken mit hunderten Titeln nicht. Also: Budget
  festlegen (Vorschlag 32 KB / 400 Knoten), darüber Lazy-Loading über `MENU_PAGE_REQ`.
- **`uid_counter`-Disziplin.** PiDrive baut den Baum bei jedem `menu_rev`-Inkrement neu,
  und das passiert bei jedem BT-/WiFi-Ereignis. Der Zähler darf sich nur erhöhen, wenn
  sich die **UID-Menge** ändert, nicht bei jedem Rebuild — sonst zieht das Auto
  permanent die Liste neu. Das ist im PiDrive-Auftrag (M1) schon so festgelegt; hier als
  Contract-Anforderung spiegeln.

---

## 5. Was ausdrücklich nicht zu tun ist

- **Keine Firmware, kein `main/`, keine `components/`** — A17 ist offen, und die
  Komponentenstruktur hängt daran.
- **Keine byte-genauen PDAP-Structs.** V1.1 hält das bewusst offen
  (`OFFENE-PUNKTE.md` Abschnitt C); mit dem möglichen Menü-Kanal gilt das noch mehr.
- **Keine AVRCP→Trigger-Mapping-Tabelle festschreiben** — das ist Ergebnis von Phase 0.
- **Nicht in `pidrive` arbeiten.** Die PiDrive-Seite hat einen eigenen Auftrag und eine
  eigene Instanz. Hier nur der Contract und Verweise.
- **Keine Bluedroid-Patches vorbereiten,** solange A17 offen ist.

---

## 6. Fragen an den Eigentümer

| # | Frage | Warum sie jetzt wichtig ist |
|---|---|---|
| Q1 | Ist ein Fahrzeugtermin für Phase −1 absehbar? | Phase −1 blockiert A17 und A18; ohne sie läuft die Planung ins Spekulative |
| Q2 | BTstack-Lizenz (frei nur nicht-kommerziell) akzeptabel? | Entscheidet A17 mit; bei „nein" bleibt nur Bluedroid + Patch und damit S1 als Maximum |
| Q3 | Wenn Browsing nicht möglich ist — bleibt das Gateway-Projekt trotzdem gewollt (nur Stabilität, Display per BTstack-Metadaten)? | Bestimmt, ob Phase −1 ein Gate oder nur eine Wegwahl ist |
| Q4 | Soll der ESP zusätzlich AVRCP-**Controller** können (für den Fall, dass später doch ein Handy direkt am ESP hängt)? | Beeinflusst A17 und die SDP-Rollen; BTstack könnte beides, Bluedroid nicht sinnvoll |
| Q5 | Zielhardware endgültig WROOM, oder WROVER/PSRAM einplanen? | Bei S3 + Menübaum + BTstack + WiFi wird der Heap enger als in L4 angenommen |

---

## 7. Reihenfolge

```
1. AVRCP-MOEGLICHKEITEN.md schreiben              (Grundlage, ohne Hardware)
2. PHASE-0-MESSPLAN.md schreiben, Phase −1 davor  (verweist auf pidrive G1/G2)
3. OFFENE-PUNKTE.md: A1 entschärfen, A17–A20, R18–R23
4. PFLICHTENHEFT.md: §2.2, §2.9, §2.10, §2.3, §2.8, neu §2.24, §2.20
5. BETRIEBSMODI.md Weg E/F · KONZEPT.md §1.2/§1.3/§1.5 · STATE.md · README.md
6. REVIEW-V1.2.md: Befunde F1–F5 als Nachzüge dokumentieren (wie V1.1 es für V1.0 tat)
7. Warten auf Phase −1 → dann A17 entscheiden → dann erst Komponentenstruktur
```

Schritte 1–6 sind reine Dokumentationsarbeit und brauchen weder Hardware noch Fahrzeug.
Schritt 7 ist der Punkt, an dem aus Ideenfindung Planung wird.
