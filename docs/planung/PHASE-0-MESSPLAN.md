# Phase −1 / Phase 0 — Messplan

**Dokumentstatus:** Planung V1.2  
**Anlass:** [AUFTRAG-CURSOR-2.md](AUFTRAG-CURSOR-2.md)  
**Kontext:** Pflichtenheft §2.3 beschrieb Phase 0 bisher nur als ESP-Analyzer. Die architekturentscheidende Messung geht **früher und billiger** mit dem Pi.

---

## 1. Übersicht

| Phase | Wer | Was | Blockiert |
|-------|-----|-----|-----------|
| **Phase −1** (neu) | Pi, BlueZ, `btmon` | Öffnet NBT Evo den Browsing-Kanal? Sendet es `SetBrowsedPlayer`/`GetFolderItems`? SDP-Feature-Bits (59/60/65)? Textzeilen im Fahrzeug? Welcher AVRCP-Eingang? | **A17, A18, S3** |
| **Phase 0** | ESP | Pass-Through-PDUs, Timing, Codec-Negotiation, Disconnect-Reasons über Zündzyklen | Mapping-Tabelle |
| **Coexistence-Gate** | ESP | wie Pflichtenheft §2.4, **plus** Eigenheiten des gewählten Host-Stacks (nach A17; bei BTstack nicht von Bluedroid ableiten) | alles Weitere |

Phase −1 und Phase 0 sind **keine Firmware-Startfreigabe**. Bis A17 entschieden ist: keine `main/`-/`components/`-Anlage (Auftrag §5).

---

## 2. Phase −1 — Pi / BlueZ / btmon

### 2.1 Zweck

BlueZ bewirbt target-seitig Browsing im SDP-Record (`avrcp_tg_record(browsing)` inkl. Additional Protocol Descriptor / AVCTP-Browsing-PSM), beantwortet aber nur einen schmalen Scope (Media Player List) und lehnt Virtual Filesystem / Search / Now Playing mit `AVRCP_STATUS_INVALID_SCOPE` ab. Viele Browse-PDUs haben target-seitig gar keinen Handler (F5).

**Konsequenz:** Wenn das BMW Browsing kann, **wird es es versuchen** — messbar mit vorhandener Hardware, ohne ESP-Firmware. Der Punkt, an dem BlueZ aufgibt, ist genau die Lücke, die ein eigener AVRCP-Target-Stack füllen würde.

Zusätzlich: Ob die drei Metadata-Zeilen im Display ankommen, entscheidet, ob S1 überhaupt als Produktziel trägt (Druck auf Stack wegen F1).

### 2.2 Durchführung

Liegt im **`pidrive`-Repo** (Arbeitspakete G1/G2), inkl.:

- `tools/bmw_avrcp_probe.sh` (oder Nachfolger)
- vorab festgeschriebenem Interpretationsschlüssel

Dieses Repo verlinkt nur und dokumentiert die **Entscheidungswirkung**. Keine Doppelpflege der Probe-Skripte hier.

### 2.3 Messfragen (Checkliste)

| ID | Frage | Evidenz |
|----|-------|---------|
| −1.1 | Wird PSM 0x001B (Browsing) vom BMW geöffnet? | `btmon` / L2CAP |
| −1.2 | Kommen `SetBrowsedPlayer`, `GetFolderItems`, evtl. `ChangePath`? | PDU-Log |
| −1.3 | Welche SDP-Feature-Bits setzt / erwartet das Fahrzeug (59 Browsing, 60 Searching, 65 Now Playing)? | SDP dump |
| −1.4 | Erscheinen Title/Artist/Album der BlueZ-MPRIS-Strecke im iDrive? | Sichtprüfung + Log |
| −1.5 | Welches Pass-Through-Subset (Pressed/Released, Vol, FF/REW)? | wie bisherige PiDrive-Beobachtung, hier nur bestätigen |

### 2.4 Entscheidungswirkung

| Ergebnis | Folge |
|----------|--------|
| **Kein** Browsing-Kanal | S3 gestrichen; Zielbild bleibt **S1**; Stack-Frage entspannt sich (Bluedroid + Metadaten-Patch wird vertretbar, BTstack weiterhin Option für saubere Meta-API) |
| Browsing-Kanal mit `SetBrowsedPlayer` (o. ä.) | **S3 = Zielbild**; BTstack wird Standardoption; PDAP braucht Menü-Kanal (**A18**) |
| Metadata erscheint **nicht** | S1 gefährdet; ohne Target-Meta-Fähigkeit (F1) ist Gateway-Display tot — A17/Q3 eskalieren |
| Metadata erscheint | S1-Minimum bestätigt; Druck auf Stack bleibt wegen fehlender Bluedroid-TG-API |

Eigentümerfragen Q1–Q3 in [AUFTRAG-CURSOR-2.md](AUFTRAG-CURSOR-2.md) §6 bestimmen, ob Phase −1 ein **Gate** (Projektstopp ohne Browsing) oder nur eine **Wegwahl** ist.

---

## 3. Phase 0 — ESP-Analyzer

Unverändert im Kern gegenüber Pflichtenheft §2.3, aber **nach** Phase −1 und idealerweise **nach** A17 (damit Analyzer auf dem finalen Stack läuft; ein früher Bluedroid-Probe ist optional und ersetzt das Gate nicht).

**Ziel:** konkretes BMW-Verhalten vermessen, bevor die Audio-Pipeline fertig ist.

**Anforderungen an den ESP (wenn Firmware startet):**

- Discovery, Pairing, Connect zum BMW
- Vollständiges Logging eingehender AVRCP-PDUs (Pass-Through, Register Notification, Get Element Attributes, Volume, ggf. Browsing falls S3)
- Timing (Pressed/Released), Reihenfolge, Parameter
- A2DP-Aufbau und Codec-Negotiation
- RSSI, Disconnect-Reasons, Connection-Interval
- WebUI + exportierbare Logs (JSON/CSV)
- Self-Test (Testton ESP → BMW)

**Exit-Kriterium:** Protokollierung über mehrere Zündzyklen und Fahrsituationen → Ableitung der Mapping-Tabelle AVRCP→PDAP-Events. Die Tabelle wird **nicht** vorab festgeschrieben.

---

## 4. Coexistence-Gate

Wie [PFLICHTENHEFT.md](PFLICHTENHEFT.md) §2.4:

1. STA + A2DP ≥ 30 min  
2. SoftAP + A2DP ≥ 30 min  
3. Optional APSTA + A2DP  

**Zusatz V1.2:** Gate mit dem **gewählten** Host-Stack wiederholen. BTstack-ESP32-Port nutzt VHCI; bekannte Controller-Hinweise im Port-README → Ergebnis von Bluedroid **nicht** übertragbar (R21).

---

## 5. Reihenfolge (Planung → Messung → Bau)

```
Phase −1 (Pi)  →  A17/A18 entscheiden  →  Komponentenstruktur
                 →  Phase 0 Skeleton (ESP)
                 →  Coexistence-Gate
                 →  Mapping / Audio / PDAP …
```

Details Ausbaustufen und Stack: [AVRCP-MOEGLICHKEITEN.md](AVRCP-MOEGLICHKEITEN.md).  
Offene Punkte: [OFFENE-PUNKTE.md](OFFENE-PUNKTE.md) A17–A20.
