# Hardware: Pegelwandler, Stromversorgung und Verdrahtung

🇬🇧 [English version](README.md)

Lies zuerst [../safety.md](../safety.md) (Englisch). Du arbeitest neben einem 12-V-Netzteil und einem Gerät, das Brennstoff verbrennt.

![Schaltplan](../images/schematic.svg)

## Was die Platine macht

Heizung und Controller sprechen über **eine einzige Datenleitung** (die blaue Ader des 3-adrigen Controller-Kabels). Die Leitung ist ein Open-Drain-Bus mit Pull-up: Teilnehmer ziehen sie nur nach unten. Ein ESP32-Pin verträgt höchstens 3,3 V, deshalb sitzt dazwischen ein kleiner Pegelwandler aus zwei MOSFETs:

- **Empfangen (Q2):** Der Bus liegt am Gate von Q2. Bus high → Q2 leitet → GPIO16 wird nach unten gezogen. Bus low → Q2 sperrt → R1 zieht GPIO16 auf 3,3 V. Das Signal ist also **invertiert** (die Firmware berücksichtigt das).
- **Senden (Q1):** GPIO17 high → Q1 leitet → der Bus wird über R3 nach unten gezogen. GPIO17 low → Q1 sperrt → der Bus ist frei.
- **R5** (10 kΩ nach 5 V) ist der Pull-up für den Bus auf der Platinenseite.

Es wird nur die DATA-Ader angezapft. Die rote (5 V) und die schwarze (GND) Ader des Controller-Kabels werden **nicht** benutzt. Masse ist über das gemeinsame 12-V-Netzteil von Heizung und XL4015 verbunden.

![Verdrahtungsübersicht](../images/wiring-overview.svg)

## Teileliste

Bestellnummern stehen absichtlich nicht drin (sie ändern sich, und falsche Nummern sind schlimmer als keine). Such nach der Typbezeichnung.

| Anz. | Bez. | Teil | Hinweis |
|---|---|---|---|
| 1 | – | ESP32-Board, **ESP32-WROOM-32D** (PlatformIO/ESPHome-Board `esp32dev`) | Kein WROVER/PSRAM-Board: dort sind GPIO16/17 intern belegt. |
| 2 | Q1, Q2 | **BS170**, N-Kanal-MOSFET, TO-92 | Pinbelegung **Drain – Gate – Source** (flache Seite zu dir, Beine nach unten, von links nach rechts). Beim 2N7000 ist es Source–Gate–Drain, also anders. |
| 1 | R5 | 10 kΩ, 1/4 W | Bus-Pull-up nach 5 V |
| 2 | R1, R3 | 1 kΩ, 1/4 W | R1: Pull-up GPIO16 nach 3V3; R3: Serienwiderstand TX zum Bus |
| 2 | R2, R4 | 100 Ω, 1/4 W | Serienschutz an GPIO16 und GPIO17 |
| 1 | – | **XL4015**-Abwärtswandler (einstellbar) | 12 V → **5,0 V**; braucht am Eingang etwa 7 V oder mehr |
| 1 | – | Lochrasterplatine, Pad pro Loch, 2,54 mm | der Prototyp nutzte 50 × 100 mm (Reichelt H25PR050) |
| – | – | Draht, Lötzinn, Schrumpfschlauch | |
| – | – | Wire-Tap (IDC-„Kabeldieb“) passend zur dünnen Controller-Ader, **oder** einen Abzweig an die blaue Ader löten | |
| – | – | Gehäuse, Zugentlastung für die Kabel | beliebiges trockenes, nicht brennbares Gehäuse, in das die Platine passt |

Werkzeug: Lötkolben, **Multimeter** (zwingend), kleiner Seitenschneider/Abisolierer, 12-V-Netzteil oder die Versorgung der Heizung zum Testen.

## Netzliste (Punkt-zu-Punkt auf Lochraster)

| Von | Nach |
|---|---|
| +5 V (XL4015 Ausgang) | R5 Pin 1; ESP32 **VIN** |
| R5 Pin 2 | **DATA**-Knoten |
| DATA-Knoten | Q2 **Gate**; R3 Pin 1; blaue Ader des Controller-Kabels |
| R3 Pin 2 | Q1 **Drain** |
| Q1 Gate | R4 Pin 2; R4 Pin 1 → ESP32 **GPIO17** |
| Q1 Source | GND |
| Q2 Drain | R1 Pin 1; R2 Pin 1 |
| R1 Pin 2 | ESP32 **3V3**-Pin |
| R2 Pin 2 | ESP32 **GPIO16** |
| Q2 Source | GND |
| GND | XL4015 GND (OUT−), ESP32 GND; XL4015 IN− geht an Minus des 12-V-Netzteils, das auch die Heizung nutzt |

Auf der Platine gibt es keinen Spannungsregler: Das ESP32-Board erzeugt seine 3,3 V selbst, der 3V3-Pin wird für R1 benutzt. Das Layout ist frei, Leitungen kurz halten. Eine Massefläche ist nicht nötig.

## Schritt 1: XL4015 auf 5,0 V einstellen, *bevor* du irgendetwas anderes anschließt

Ein nicht eingestellter Abwärtswandler gibt am Ausgang eine hohe Spannung weiter und **zerstört den ESP32**.

1. Nur den 12-V-Eingang (+/−) anschließen. Den Ausgang offen lassen.
2. Den Ausgang mit dem Multimeter messen.
3. Den Mehrgang-Trimmer langsam drehen, bis das Multimeter **5,0 V** zeigt (der Autor kam auf 5,01 V). Die Werkseinstellung liegt am oberen Ende, und der Trimmer braucht **viele Umdrehungen (10–20)**, bis sich der Ausgang merklich bewegt. Schau aufs Multimeter, nicht auf die Zahl der Umdrehungen.
4. Ausschalten, den Ausgang an die Platine anschließen (+5 V, GND), einschalten und die Spannung unter Last noch einmal prüfen.

## Schritt 2: Platine ohne ESP32 und ohne Heizung prüfen

- Durchgang: kein Kurzschluss zwischen +5 V und GND, keiner zwischen DATA und GND.
- BS170-Pinbelegung zweimal prüfen. Vertauschte Beine sind der häufigste Fehler.

## Schritt 3: Loopback-Test (ESP32 auf der Platine, Heizungsbus **nicht** angeschlossen)

1. Die blaue Ader noch **nicht** anschließen.
2. [`esphome/helpers/loopback-test.yaml`](../../esphome/helpers/loopback-test.yaml) flashen (eigene `secrets.yaml` nötig). Beim ersten Flashen über USB nicht gleichzeitig 5 V vom XL4015 auf VIN geben, außer du weißt, dass dein Board USB und VIN trennt. Nur über USB versorgen, danach auf den XL4015 wechseln und später per OTA flashen.
3. In Home Assistant (oder im Web-Log) **TEST Pulse 1s** drücken:
   - *TEST RX Q2* = **OFF** im Ruhezustand, **ON** während des 1-s-Pulses → Q1 und Q2 funktionieren.
   - Dauerhaft ON → Q2 schaltet nicht (Pinbelegung, Lötstelle, R5 fehlt).
   - Dauerhaft OFF, keine Änderung beim Puls → Q1 schaltet nicht.

## Schritt 4: an den Heizungsbus anschließen

1. Heizung **im Standby**, Controller wie bisher angeschlossen.
2. Gleichspannung zwischen blauer Ader und GND mit dem Multimeter messen. Beim Autor: im Ruhezustand etwa **4,85 V**. Solange Frames auf dem Bus laufen, flackert das Multimeter bis etwa 2,35 V herunter (es mittelt die 4/8-ms-Pulse, das ist also ein Mittelwert und nicht der logische Low-Pegel). Andere Projekte berichten für diese Art Heizung von etwa 4 V und einem verrauschten Bus. **Bei mehr als etwa 6 V stoppen** und den ESP32 nicht anschließen.
3. Die blaue Ader anzapfen (IDC-Klemme oder angelöteter Abzweig) und mit dem DATA-Knoten verbinden. Den Abgriff kurz und isoliert halten.
4. [`esphome/vevor-8kw-heater.de.yaml`](../../esphome/vevor-8kw-heater.de.yaml) flashen. Erwartet: *Heizung Bus aktiv* = an, *Heizung Status* = Standby, *Heizung Spannung* ≈ 12–13 V, keine „unlesbar“-Meldungen in *Heizung Ereignis*.
5. Werden Frames nicht dekodiert, [`esphome/helpers/raw-capture.yaml`](../../esphome/helpers/raw-capture.yaml) benutzen und die Rohwerte mit der [Protokollbeschreibung](../protocol/protocol.de.md) vergleichen. Das Controller-Display muss mit dem Abgriff weiterhin normal funktionieren.

## Hinweise

- GPIO16/GPIO17 und die Polarität (RX invertiert, TX aktiv-high) sind in der Firmware festgelegt. Wenn du Pins änderst, ändere `remote_receiver` und `remote_transmitter` beide.
- Die Gate-Schwelle des BS170 ist bis etwa 3 V spezifiziert. Das getestete Gerät liegt im Ruhezustand bei etwa 4,85 V, also deutlich darüber. Eine viel niedrigere Ruhespannung würde Q2 nicht sicher schalten.
- Die Platine in ein Gehäuse setzen, alle Leitungen mit Zugentlastung. Der XL4015 kann warm werden; er versorgt nur den ESP32 und bleibt weit unter seinen Grenzen.
- Das Kabel des ESP32-Boards nicht entlang von Abgas oder Warmluftaustritt der Heizung verlegen.
