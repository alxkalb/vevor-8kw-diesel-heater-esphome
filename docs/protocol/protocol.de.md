# Bus-Protokoll (VEVOR 8 kW Heizung, Controller mit blauem LCD)

🇬🇧 [English version](protocol.md)

Reverse-Engineering aus Captures mit dem `remote_receiver` von ESPHome an einem Gerät. Alles hier beschreibt, **was beobachtet wurde**; nur vermutete Dinge sind mit *(Hypothese)* markiert. Den Verifizierungsstand findest du in [../status/status.md](../status/status.md) (Englisch).

## Das ist kein UART

Andere Open-Source-Projekte für Vevor-Heizungen (zum Beispiel die in der Haupt-README genannten) sprechen **4800 Baud UART** mit dem Controller. Diese Heizung nicht. Ihre einzelne Datenleitung trägt einen langsamen Pulscode. Ein ESPHome-`uart:` mit 4800 oder 25000 Baud dekodiert hier nichts Brauchbares.

## Physikalische Ebene

- Eine Leitung, Open-Drain mit Pull-up, Ruhe = high. Teilnehmer ziehen den Bus nach unten.
- Über den Pegelwandler sieht der ESP32 den Bus **invertiert**: Bus high = GPIO16 low.
- An diesem Gerät mit dem Multimeter an der DATA-Ader gemessen: Ruhe etwa **4,85 V**. Solange Frames auf dem Bus laufen, flackert die Anzeige zwischen 4,85 V und etwa 2,35 V. Beide Werte waren mit und ohne angeschlossene Pegelwandler-Platine gleich. Ein Multimeter löst die 4/8-ms-Pulse nicht auf und zeigt einen Mittelwert, 2,35 V ist also kein logischer Pegel. Der echte Low-Pegel wurde nicht separat gemessen. Andere Projekte berichten für diese Art Heizung von etwa 4 V und einem verrauschten Bus (3,3–6 V).

## Bitkodierung

Jedes Bit ist eine 12-ms-Zelle, **MSB zuerst**. Ein Bit ist „Bus high, dann Bus low“:

| Bit | Bus high | Bus low | `remote_receiver`-Rohwerte (invertiert: − = Bus high) |
|---|---|---|---|
| `1` | 8 ms | 4 ms | `-8000, +4000` |
| `0` | 4 ms | 8 ms | `-4000, +8000` |

Toleranz des Decoders in der Firmware: langer Teil 6,5–9,5 ms, kurzer Teil 3,0–5,2 ms. `idle: 25ms` trennt Frames, `filter: 10us` entfernt Störspitzen.

## Frames und Timing

Einmal pro Sekunde, in dieser Reihenfolge:

| Teil | Richtung | Länge | Dauer |
|---|---|---|---|
| **A-Frame** | Controller → Heizung | 8 Bit | 96 ms |
| Lücke (ein Low-„Startpuls“) | | | etwa 30 ms |
| **B-Frame** | Heizung → Controller | 16 Bit | 192 ms |
| Ruhe | | | etwa 680 ms |

Wenn der ESP32 einen Frame sendet, hält er den Bus zuerst 30 ms auf low (der Startpuls) und sendet dann die Bits. In der Firmware ist das der erste Wert `30000` in der `transmit_raw`-Liste.

## A-Frames (Controller → Heizung)

| Code | Binär | Bedeutung |
|---|---|---|
| `A6` | 1010 0110 | Leerlauf / Keep-alive, jede Sekunde |
| `A2` | 1010 0010 | **Ein/Aus-Umschalter** |
| `A3` | 1010 0011 | Stufe hoch |
| `A4` | 1010 0100 | Stufe runter |

- Das obere Nibble ist immer `A`. Die Firmware ignoriert 8-Bit-Frames, die nicht mit `A` beginnen.
- `A2` von der Controller-Taste, von der 433-MHz-Fernbedienung und vom ESP32 sieht auf dem Bus identisch aus.
- Ein/Aus ist ein **Umschalter**. Die Heizung sagt dem Sender nicht, ob der Befehl „ein“ oder „aus“ war; der Sender muss den Zustand kennen. Die Firmware sendet ihn deshalb nur, wenn der B-Frame den passenden Zustand zeigt.
- Im Standby erzeugen die Controller-Tasten *Hoch, Runter, OK, Gang* keine zusätzlichen Frames auf dem Bus.
- Die Heizung **merkt sich die letzte Stufe**; ein Start nutzt sie.

## B-Frames (Heizung → Controller), 16 Bit

Als Hex geschrieben `HHLL`.

### Standby

`60xx`: Das untere Byte ist die **Versorgungsspannung in Volt** (ganzzahlig). `0x0C` und `0x0D` wechseln sich ab (12 V und 13 V); die Firmware akzeptiert `0x0A`–`0x1F`.

### Aktiv (Start, Betrieb)

Das untere Byte ist `0x3F + Stufe`: `0x40`–`0x45` = Stufe 1–6. Das obere Byte beschreibt die Phase. *(Hypothese, passt zu allen Captures)* Die Bits des oberen Bytes sind: `0x08` Lüfter, `0x04` Pumpe, `0x01` Glühkerze. Beobachtete Startsequenz bei Stufe 3 (unteres Byte `42`):

| B-Frame | Bits im oberen Byte | Gedeutete Phase (Firmware-Text) |
|---|---|---|
| `6042` | keine | Start angefordert |
| `6842` | Lüfter | Anlauf |
| `6942` | Lüfter + Glühkerze | Vorgluehen |
| `6D42` | Lüfter + Pumpe + Glühkerze | Zuenden |
| `6D42` / `6942` im Wechsel | | Pumpenpulse, während das Vorglühen weiterläuft |
| `6C42` | Lüfter + Pumpe | Laeuft |

### Abkühlen

Unteres Byte `C0`: Abkühlphase. Nach einem normalen Stopp beobachtet: `64C0`/`65C0` → `6CC0`/`6DC0` → `69C0` → `68C0` → `600C` (zurück im Standby, 12 V). `61C0` und `60C0` nach einem sehr kurzen Lauf, der direkt im Standby endete.

Gemessene Abkühlzeiten (ein Gerät): etwa 1 min 30 s nach einem Lauf von etwa 1 min 55 s; etwa 5 min 32 s nach einem Lauf von etwa 3 min. Nach einem sehr kurzen Lauf kühlte die Heizung gar nicht ab.

### Störung

Das obere Nibble `8` kennzeichnet einen Fehler-Frame. Beobachtet bei leerem Tank (Display: E08):

`8CC9` → `89C9` → `88C9` → `8009` (etwa 2 min 20 s vom ersten Fehler-Frame bis zum Fehler-Standby)

- Unteres Byte & `0x3F` = `0x09` in diesem Fall. *(Hypothese)* Die angezeigte E-Nummer ist dieser Wert minus 1, was nur zu dieser einen Beobachtung passt. Die Firmware zeigt deshalb `E-08 (?)` mit Fragezeichen.
- Mit gesetztem `C0` im unteren Byte (`8CC9`, `89C9`, `88C9`) kühlt die Heizung noch ab. `8009` ist der Fehler-Standby.
- Die Störung bleibt bis zum nächsten Start bestehen. EIN im Fehler-Standby (`8009`) löscht sie, und die Heizung startet normal (verifiziert).

Zusätzlich gibt es in der Firmware eine zweite Störungserkennung: Endet ein Betrieb (unteres Byte nicht mehr `0x40`–`0x45`) und liegt davor mindestens 2 Frames lang kein `A2`, wird „Fehler!“ angezeigt. Das erfasst ein Abschalten durch die Heizung selbst.

## Befehle senden: die Regeln, die funktioniert haben

1. Nie senden, während ein anderer Frame auf dem Bus ist. Eine Kollision zerstört beide Frames; die Heizung antwortet auf einen kaputten Frame nicht. In der nächsten Sekunde erneut versuchen.
2. Das funktionierende Fenster liegt **etwa 100 ms nach einem regulären B-Frame**, der auf einen Controller-A-Frame folgte (nicht nach der B-Antwort auf den eigenen Befehl des ESP).
3. Startpuls (Bus low) 30 ms, dann 8 Bits, MSB zuerst.
4. Die Heizung beantwortet einen zusätzlichen A-Frame etwa 20–30 ms später mit einem B-Frame. Dieser B-Frame zeigt den neuen Zustand (zum Beispiel Stufe + 1).
5. Stufenwechsel: ein `A3`/`A4` pro Schritt. Die Firmware merkt bis zu 6 Schritte vor, einen pro freiem Fenster, und prüft bei jedem Schritt die Grenzen (1–6).

## Rohbeispiel (Frame `A2`, wie in `esphome/helpers/raw-capture.yaml`)

```
30000,                  # Startpuls: Bus low 30 ms
-8000,4000, -4000,8000, # 1 0
-8000,4000, -4000,8000, # 1 0
-4000,8000, -4000,8000, # 0 0
-8000,4000, -4000,8000  # 1 0   -> A2 = 1010 0010
```

(Vollständiger Frame, zur Lesbarkeit zwei Bits pro Zeile. In der echten `code:`-Liste ist alles eine flache Zahlenliste.)

## Vergleich mit kellerman/vevor_pwm

[kellerman/vevor_pwm](https://github.com/kellerman/vevor_pwm) (früher veröffentlicht) beschreibt denselben Bus-Typ für „Vevor PWM“-Heizungen: 8-Bit-Befehle an die Heizung, 16-Bit-Statuswörter von der Heizung, 30 ms Startpuls, etwa 12 ms pro Bit. Vergleicht man seine README mit den hier gesehenen Frames, **stimmen die Werte überein, wenn man jedes Bit invertiert** (seine Polaritätskonvention ist das Gegenteil der in diesem Dokument benutzten). Die Tabelle ist mein eigener Vergleich anhand seiner README. Sie wurde nicht an seiner Hardware getestet, und seine Bibliothek wurde hier nicht ausgeführt.

| Dieses Projekt (hier dekodiert) | vevor_pwm-README | Bedeutung dort |
|---|---|---|
| A-Frame `A6` | Befehl `0x59` (Standardbits) | Leerlauf |
| A-Frame `A2` | `0x5D` (Bit 2 gesetzt) | Ein/Aus |
| A-Frame `A4` | `0x5B` (Bit 1 gesetzt) | „Power level toggle“ (zyklisch 6 → 1 → 6; hier: Stufe runter) |
| A-Frame `A3` | `0x5C` | nicht in seiner Tabelle (hier: Stufe hoch) |
| `600C` … `6009`, `600F` | `0x9FF3` … `0x9FF6`, `0x9FF0` | Leerlauf/Standby mit Batteriespannung (hier: 12 V … 9 V, 15 V) |
| `6845`, `6945`, `6D45` | `0x97BA`, `0x96BA`, `0x92BA` | Startsequenz (seine Werte gelten für Stufe 6; hier: Lüfter, Vorglühen, Zündung) |
| `6C40` … `6C45` | `0x93BF` … `0x93BA` | aktiv, letztes Nibble `F`…`A` = Stufe 1…6 |
| `6CC0` | `0x933F` | Abschaltung läuft |
| `69C0` | `0x963F` | „Clearing flame“ (Flamme löschen) |
| `68C0` | `0x973F` | Abkühlen |
| `0x8000` … `0x8FFF` | `0x7000` … `0x7FFF` | Fehlerzustände |

Was das zum Gesamtbild beiträgt: Das Abkühlen, das hier als eine Phase gesehen wird (unteres Byte `C0`), ist dort eine Folge aus drei benannten Stufen, und eine unabhängige Quelle beschreibt das Stufen-Nibble und die Versorgungsspannung genauso. Was es nicht klärt: Die Bedeutung der Lüfter-/Pumpen-/Glühkerzen-Bits und der Hinweis zur E-Nummer bleiben Hypothesen. Seine README nennt außerdem ein „Ready“-Statuswort (`0xFA6C`), das in den Captures dieses Projekts nicht gesehen wurde.

### Was andere berichten (aus zweiter Hand, hier nicht verifiziert)

Quellen: README und Quelltext von [vevor_pwm](https://github.com/kellerman/vevor_pwm) sowie der [Home-Assistant-Forumthread](https://community.home-assistant.io/t/vevor-diesel-heater-control-development-in-progress/832159) (Beiträge von kellerman, Emvace und Cajda91, 2026).

- **Polaritätskonvention:** vevor_pwm dekodiert ein Bit nach der Breite des GPIO-Low-Abschnitts, der über den invertierenden Pegelwandler der Bus-*High*-Zeit entspricht: etwa 4 ms = Bit 1, etwa 8 ms = Bit 0. In der Konvention dieses Dokuments sind das `0` und `1`. Die Signale auf der Leitung sind dieselben, nur die Bitwerte sind andersherum benannt.
- **Bitzelle:** Ihr Sendecode nutzt etwa 12,1 ms (4,0 + 8,1 ms und 8,0 + 4,1 ms). Dieses Projekt sendet 12-ms-Zellen. Beides funktioniert.
- **Vorpuls und Ruhe:** Ihr Empfänger erwartet vor dem 30-ms-Startpuls eines 16-Bit-Frames einen kurzen Vorpuls (etwa 0,8–3 ms) und setzt sich nach mehr als etwa 80 ms ohne vollständigen Frame zurück. Hier nicht separat geprüft.
- **Ein anderes Gerät:** Eine CY-32-Heizung (Emvace, Cajda91) zeigt im Leerlauf `0xDFF3`, wo das erste vevor_pwm-Gerät `0x9FF3` zeigt. Invertiert ist das `200C` statt `600C`: dasselbe Spannungsbyte, aber Bit `0x40` des oberen Bytes ist anders. Das erwartete Muster von `9Fxx` auf `DFxx` zu ändern, ließ dort den Empfang funktionieren. **Diese Firmware verlangt im Standby das obere Byte `0x60`** und würde ein solches Gerät nicht erkennen (Zustand „unbekannt“, EIN wird ignoriert).
- **Fehlerwort an einem anderen Gerät:** kellerman nennt `7FF3` als Fehlerwort (invertiert: `800C`) und `9FF3` als Leerlauf. Hier endete die Störung „Tank leer“ in `8009`. Ob das eine andere Störung, ein anderes Modell oder eine andere Auslesung ist, ist unbekannt. Es zeigt, dass das untere Byte eines Fehler-Standbys nicht zwingend eine einfache Fehlernummer ist.
- **Stufenbefehle:** Mit der vevor_pwm-Bibliothek läuft der Leistungsbefehl zyklisch 6 → 1 → 6 (im Thread als anders als beim Originalcontroller beschrieben). Hier sind `A3` und `A4` getrennte Frames für Stufe hoch und runter.
- **Startbefehl an der CY-32:** Cajda91 berichtet, die Heizung zum Laufen gebracht zu haben, indem er die Befehlsbits 4 und 6 änderte (ohne Details). Die Bedeutung der Befehlsbits kann sich zwischen Controller-Varianten unterscheiden.

## Unterschiede zu den UART-Projekten

| | diese Heizung | zatakon/esphome-vevor-heater (XMZ-D2) |
|---|---|---|
| Kodierung | 12-ms-Pulszelle, A- (8 Bit) und B-Frames (16 Bit), 1 pro Sekunde | UART 4800 Baud, Bytes |
| Richtung | Halbduplex auf einer Leitung | Halbduplex auf einer Leitung |
| Schnittstelle | `remote_receiver` / `remote_transmitter` | `uart:` |

## Die eigene Heizung mitschneiden

[`esphome/helpers/raw-capture.yaml`](../../esphome/helpers/raw-capture.yaml) flashen und im Log nach `Received Raw:`-Zeilen suchen. Mit der Tabelle oben vergleichen. Wenn deine Heizung 8-ms/4-ms-Zellen nutzt, ist es wahrscheinlich dasselbe Protokoll. Bitte ein Issue mit einem Log-Auszug öffnen (ohne IP/MAC/WLAN-Namen).
