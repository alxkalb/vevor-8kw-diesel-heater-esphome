# Projektstatus: was verifiziert ist und was nur vermutet wird

🇬🇧 [English version](status.md)

Stand dieses Dokuments: Erstveröffentlichung. **Ein Heizungsgerät** (VEVOR 12 V, „8 kW“, Controller mit blauem LCD) mit einem ESP32-WROOM-32D, ESPHome 2026.9.1 und 2026.10.0, Home Assistant 2026.10.

## An der Hardware verifiziert

| Was | Beleg |
|---|---|
| Pegelwandler funktioniert in beide Richtungen | Loopback-Test, Bus-Frames dekodiert |
| Dekodierung der 8-Bit-A- und 16-Bit-B-Frames | Live-Log, einmal pro Sekunde, stundenlanger Betrieb |
| Standby erkannt, Versorgungsspannung aus `60xx` gelesen | 12/13 V im Wechsel |
| Start aus dem Standby mit `A2` vom ESP32 | Heizung startet, Display folgt |
| Stopp mit `A2` im Betrieb | Heizung geht ins Abkühlen |
| Stufe hoch/runter mit `A3`/`A4` | Heizung wechselt die Stufe, B-Frame zeigt die neue Stufe |
| B-Frames der Startsequenz (`6042` … `6C42`) | bei Stufe 3 mitgeschnitten |
| Abkühlzeiten | etwa 1 min 30 s nach etwa 1 min 55 s Lauf; etwa 5 min 32 s nach etwa 3 min Lauf |
| Ein langer Lauf | etwa 2 h 52 min auf Stufe 5, bis der Tank leer war; die Heizung hat den Trockenlauf selbst abgefangen (Fehler-Frames, Abkühlen, Standby), das Display zeigte E08 |
| Fehler-Frames bei „Tank leer“ | `8CC9` → `89C9` → `88C9` → `8009` |
| **EIN aus dem Fehler-Standby (`8009`)** | löscht die Störung und startet normal; nach AUS kühlte die Heizung ab und ging in den normalen Standby |
| Controller-Display und 433-MHz-Fernbedienung funktionieren mit dem Abgriff weiter | `A2` von beiden auf dem Bus gesehen |
| Busspannung mit Multimeter an der DATA-Ader | Ruhe etwa 4,85 V; während des Datenverkehrs flackert die Anzeige zwischen 4,85 V und etwa 2,35 V, mit und ohne angeschlossene Pegelwandler-Platine (R5 ändert den Ruhepegel also nicht merklich). Ein Multimeter mittelt den Pulszug, 2,35 V ist ein Mittelwert und kein gemessener logischer Pegel |
| Entitäten-Aufräumen (die meisten Entitäten als *Diagnose* markiert) | geflasht und läuft sauber |

## Hypothesen (passen zu den Captures, sind aber nicht bewiesen)

| Was | Warum es nur eine Hypothese ist |
|---|---|
| Bits im oberen Byte: `0x08` Lüfter, `0x04` Pumpe, `0x01` Glühkerze | passt zu allen Captures; keine direkte Messung von Pumpe oder Kerze |
| Namen der Startphasen (Anlauf, Vorglühen, Zünden, Läuft) | aus dem Bitmuster oben abgeleitet |
| E-Nummer = (unteres Byte & `0x3F`) − 1 | ein Datenpunkt (E08 ↔ Wert 9). Andere als `E-xx (?)` gezeigte Nummern sind Vermutungen |
| `C0` im unteren Byte = Abkühlen | bei allen Stopps und beim Abkühlen nach Störung gesehen, aber nicht in jedem möglichen Zustand |
| Ganze Volt im unteren Byte von `60xx` | nur 12/13 gesehen; die genaue Skalierung ist nicht bekannt. Die [vevor_pwm](https://github.com/kellerman/vevor_pwm)-README (verglichen im [Protokoll-Dokument](../protocol/protocol.de.md#vergleich-mit-kellermanvevor_pwm)) beschreibt das Standby-Wort als Träger der Batteriespannung, das passt |
| Stufe = unteres Byte − `0x3F`; Abkühlen in mehreren Stufen; Fehlerbereich `0x8___` | stimmt mit derselben README überein (alle Bits invertiert); an deren Hardware nicht getestet |

## Noch nicht getestet

- Andere Fehlercodes (Überhitzung, Flammabriss, Unterspannung, ...). Ihre Frames und was die Firmware anzeigt, sind unbekannt. Gesehen wurde nur die Störung „Tank leer“ (E08), einschließlich ihres Löschens mit EIN.
- Lange Läufe auf Stufe 1 und 2; Betrieb bei Kälte; Verhalten bei instabiler 12-V-Versorgung.
- Was passiert, wenn ESP32, WLAN oder Home Assistant **während** eines Laufs ausfallen oder neu starten (erwartet: die Heizung läuft weiter, die Firmware dekodiert den Zustand nach dem nächsten B-Frame wieder; nicht getestet).
- Langzeitzuverlässigkeit (Wochen im Betrieb, z. B. Speicher- oder Timing-Drift im ESP32).
- Andere VEVOR-Modelle und andere Controller-Versionen.
- `esphome compile` der englischen Variante. In der Entwicklungsumgebung konnte nur `esphome config` (Validierung) laufen. Die Logik lief auf der Hardware in der deutsch benannten Variante [`vevor-8kw-heater.de.yaml`](../../esphome/vevor-8kw-heater.de.yaml); die englische Datei benennt nur Entitäten/Texte um und ergänzt Kommentare (maschinell verglichen).

## Bekannte Schwächen des Ansatzes

- Ein/Aus ist ein Umschalter ohne Bestätigung auf dem Bus. Eine falsche Zustandsannahme (etwa nach einem verlorenen Frame) würde in die falsche Richtung schalten. Die Firmware begrenzt das, indem sie `A2` nur sendet, wenn der letzte B-Frame zur gewünschten Aktion passt.
- Die Firmware kann die Heizung nicht stoppen, wenn ESP32, WLAN oder Home Assistant ausfallen. Siehe [../safety.de.md](../safety.de.md).
- Zustandsnamen sind Deutungen. Im Zweifel *Heizung Rohstatus* ansehen.

## Gesuchte Berichte

Wenn du ein anderes Modell testest oder von selbst eine Störung auftritt: Bitte die rohen `B=…`-Frames und die Anzeige des Displays posten. Keine Störungen absichtlich auslösen. Vorher IP/MAC/WLAN-Namen aus den Logs entfernen.
