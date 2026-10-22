# Sicherheitshinweise

🇬🇧 [English version](safety.md)

Dieses Projekt verbindet einen Mikrocontroller mit einer **Dieselheizung**. Fehler können zu Brand, Kohlenmonoxidvergiftung, Kraftstoffaustritt oder Schäden an der Heizung führen. Bau und Nutzung erfolgen **auf eigenes Risiko**. Software und Dokumentation werden ohne jede Gewährleistung bereitgestellt (siehe [LICENSE](../LICENSE)). Der Autor ist kein zertifizierter Heizungsinstallateur. Dieses Projekt steht in keiner Verbindung zu VEVOR und wurde von VEVOR nicht geprüft oder freigegeben.

## Bevor du die Heizung betreibst

- **Abgas:** Die Abgasleitung muss gasdicht ins Freie führen, die Mündung frei und fern von allem Brennbaren. Die Heizung nie mit beschädigter oder blockierter Abgas- oder Ansaugleitung betreiben.
- **Kohlenmonoxid:** Einen funktionierenden CO-Melder im Raum der Heizung und in angrenzenden Räumen verwenden. Fernstart ersetzt weder Belüftung noch die Kontrolle der Installation.
- **Brand:** Brennbares (Kraftstoffkanister, Papier, Holzspäne, Sägemehl, Lösungsmittel) von Heizung, Warmluftaustritt und Abgasrohr fernhalten. Eine Werkstatt oder Scheune ist dafür ein Hochrisiko-Ort.
- **Kraftstoffsystem:** Leitungen, Schellen und Tank auf Dichtheit prüfen. Den vom Hersteller vorgeschriebenen Kraftstoff verwenden.
- **Örtliche Vorschriften:** Einbau, Abgasführung und Betrieb von Feuerstätten sind vielerorts geregelt (Gebäude, Fahrzeuge). Halte dich an die Vorschriften an deinem Ort.

## Fernstart ist ein anderes Risiko als Starten von Hand

- **Verlass dich nicht darauf, dass ESP, WLAN oder Home Assistant die Heizung ausschalten.** Fällt eines davon aus, während die Heizung läuft, bleibt sie in ihrem letzten Zustand, bis du sie am Controller oder an der Fernbedienung ausschaltest oder der Kraftstoff leer ist. Die Firmware kann ein Abschalten nicht garantieren.
- Überlege, ob es vertretbar ist, die Heizung zu starten, wenn niemand vor Ort ist. Wenn ja, ergänze unabhängige Sicherungen (siehe unten).
- Ein Timer oder eine Automation in Home Assistant, die die Heizung nach N Stunden ausschaltet, hilft, hängt aber ebenfalls davon ab, dass das System funktioniert.
- **Die Stromversorgung nie während der Abkühlphase trennen**, außer im Notfall. Die Heizung kühlt nach einem längeren Lauf etwa 1,5 bis 5,5 Minuten ab (an einem Gerät beobachtet). Wird in dieser Zeit die Versorgung abgeschaltet, bleibt die Brennkammer heiß, und die Heizung kann Schaden nehmen.
- Erwäge eine unabhängige Hardware-Absicherung (zum Beispiel einen CO-Melder, den du hörst oder der dich benachrichtigt, einen Brandmelder, eine Versorgung, die du von einem sicheren Ort abschalten kannst).

## Elektrische Punkte

- Der Heizungsbus liegt am getesteten Gerät im Ruhezustand bei etwa 4,85 V (Multimeter). Miss deinen zuerst. **Die DATA-Ader nie direkt an einen ESP32-Pin anschließen.** Der ESP32 ist nicht 5-V-tolerant. Nutze den Pegelwandler aus diesem Projekt und teste ihn, bevor du die Heizung anschließt ([Loopback-Test](../esphome/helpers/loopback-test.yaml)).
- Den XL4015-Ausgang **mit dem Multimeter auf 5,0 V einstellen, bevor** du den ESP32 anschließt. Ein falsch eingestellter Abwärtswandler kann die volle Eingangsspannung ausgeben und zerstört den ESP32.
- Das Anzapfen des Controller-Kabels kann bei schlechter Ausführung dazu führen, dass Controller oder Heizung sich merkwürdig verhalten. Mit der Heizung im Standby testen. Den Abgriff kurz und gut isoliert halten.
- Der ESP32 simuliert nur Tastendrücke. Er berührt die eigenen Schutzfunktionen der Heizung nicht (Überhitzung, Flammabriss, Unterspannung).

## Was die Firmware leistet und was nicht

- Sie sendet Befehle nur in Zuständen, in denen sie sinnvoll sind, und nur in einem freien Zeitfenster. Das verringert Fehler, macht das System aber nicht von allein sicher.
- Die Zustandsdeutung beruht teilweise auf Hypothesen. Siehe [Status](status/status.de.md), bevor du dich bei Entscheidungen auf Texte wie „Fehler“ oder „Läuft“ verlässt.
- Es wurde nur ein Heizungsgerät getestet.
