# Kukirin G2 Pro UART Kommunikationsprotokoll - Reverse Engineering Dokumentation (V8)

## 1. Physikalische Schnittstelle

### Grundparameter
- **Protokoll**: UART (asynchron)
- **Baudrate**: 9600 bps
- **Datenformat**: 8N1 (8 Datenbits, keine Parität, 1 Stoppbit)
- **Spannung**: 3.3V TTL
- **Verbindung**: Display (Master) ↔ Motorcontroller (Slave)

## 2. Kommunikationsstruktur

### 2.1 Timing-Charakteristik
- **Zyklus-Intervall**: ~60ms
- **Wiederholrate**: ~16-17 Hz

### 2.2 Sequenz-Muster
```
[TX Burst: 3 Pakete] → [Pause: 2-5ms] → [RX Burst: 3 Pakete] → [Idle: ~45ms]
```

## 3. Paketformat

### 3.1 TX-Pakete (Display → Controller) - VOLLSTÄNDIG ENTSCHLÜSSELT
Die Analyse hat die Funktion aller grundlegenden Steuerbefehle und der Checksumme aufgedeckt.

**Paketlänge**: 20 Bytes

| Offset | Länge | Bezeichnung           | Beispielwerte         | Beschreibung & Erkenntnisse                                                                 |
|--------|-------|-----------------------|-----------------------|---------------------------------------------------------------------------------------------|
| 0x00   | 1     | Start-Marker          | 0x01                  | Feste Kennung für TX-Pakete.                                                                |
| 0x01   | 1     | Paketlänge            | 0x14                  | Immer 20 Bytes.                                                                             |
| 0x02   | 2     | Kommando-Typ          | 0x0102                | Statischer Befehls-Header.                                                                  |
| 0x04   | 1     | Fahrmodus             | 0x05/0x0A/0x0F        | Wählt die Leistungsstufe (1/2/3).                                                           |
| 0x05   | 1     | Funktions-Bitmaske    | 0x80 (Basis)          | Steuert Zusatzfunktionen via Bits: +0x40(Bremse), +0x20(Licht), +0x10(Hupe), +0x08(Blinker L), +0x04(Blinker R). |
| 0x06   | 6     | Statische Parameter   | konstant              | Unveränderte Konfigurationswerte.                                                           |
| 0x0C   | 2     | Geschwindigkeits-Profil | 0x0C19/0x0C64       | Wählt die V-Max Begrenzung (Standard/Offen). Little-Endian.                                  |
| 0x0E   | 2     | Statischer Parameter   | 0x01A4                | Unveränderter Konfigurationswert.                                                           |
| 0x10   | 2     | Throttle-Sollwert     | 0x0000-0x03E8         | Gashebel-Position (Little-Endian). Maximalwert wird vom Fahrmodus skaliert.                  |
| 0x12   | 2     | Checksumme            | variiert              | CRC-16/MODBUS der ersten 18 Bytes (Little-Endian). Siehe Sektion 4.2.                        |

### 3.2 RX-Pakete (Controller → Display) - VOLLSTÄNDIG ENTSCHLÜSSELT
Alle Felder, inklusive der Checksumme, sind nun identifiziert.

**Paketlänge**: 16 Bytes

| Offset | Länge | Bezeichnung           | Beispielwert          | Beschreibung & Erkenntnisse                                                                 |
|--------|-------|-----------------------|-----------------------|---------------------------------------------------------------------------------------------|
| 0x00   | 1     | Start-Marker          | 0x02                  | Antwort-Kennung.                                                                            |
| 0x01   | 1     | Paketlänge            | 0x0E                  | 14 Bytes Nutzdaten.                                                                         |
| 0x02   | 1     | Status-Typ            | 0x01                  | Antworttyp.                                                                                 |
| 0x03   | 1     | Status-Flag           | 0x00 / 0x80           | 0x80 wird einmalig beim Start als "Boot-Acknowledged" gesendet. Sonst 0x00.                  |
| 0x04   | 1     | System-Status         | 0xC0 / 0xE0           | Meldet den Bremsstatus. 0xC0 (Normal). 0xE0 wenn Bremse aktiv ist (Bit 5 wird gesetzt).      |
| 0x05   | 3     | Unbekannt             | 0x000000              | Bisher konstant bei 0x000000.                                                               |
| 0x08   | 2     | Geschwindigkeit       | 0x0DAC (Idle)         | Der eigentliche Geschwindigkeitswert. 2-Byte Big-Endian, invers proportional zur Raddrehzahl. |
| 0x0A   | 2     | Unbekannt             | 0x0000                | Unbekannt.                                                                                  |
| 0x0C   | 1     | Status Bremse/Reku    | 0x6C/0x4C             | Wert ändert sich von 0x6C zu 0x4C, wenn die Bremse aktiv ist.                               |
| 0x0D   | 1     | Checksumme            | variiert              | CRC-8 Prüfsumme der ersten 13 Bytes (0x00-0x0C). Siehe Sektion 4.3.                         |
| 0x0E   | 2     | Echo/Prüfung          | 0x020E                | Wiederholung des Paket-Headers (Start-Marker & Paketlänge).                                  |

## 4. Protokoll-Analyse & Steuerungslogik

### 4.1 Steuerungslogik
- **Befehl Senden (TX)**: Das Display sendet kontinuierlich Pakete, die den Zustand aller Bedienelemente abbilden.
- **Ausführung (Controller)**: Der Controller führt die Befehle aus und begrenzt die V-Max intern basierend auf dem gewählten Geschwindigkeits-Profil.
- **Rückmeldung (RX)**: Der Controller sendet Statuspakete zurück, die Messwerte (Geschwindigkeit) und Bestätigungen (Bremse) enthalten.

### 4.2 Checksummen-Berechnung (TX)
Die 2-Byte-Prüfsumme wird über die ersten 18 Bytes des Pakets (Offset 0x00 bis 0x11) berechnet und im Little-Endian-Format angehängt.

- **Algorithmus**: CRC-16/MODBUS
- **Polynom**: 0x8005
- **Initialwert**: 0xFFFF
- **Input reflektiert**: Ja
- **Output reflektiert**: Ja
- **XOR Out**: 0x0000

### 4.3 Checksummen-Berechnung (RX)
Die 1-Byte-Prüfsumme wird über die ersten 13 Bytes des Pakets (Offset 0x00 bis 0x0C) berechnet und an Position 14 (Offset 0x0D) angehängt.

- **Algorithmus**: CRC-8
- **Polynom**: 0x31 (x⁸ + x⁵ + x⁴ + 1)
- **Initialwert**: 0x00
- **Input reflektiert**: Nein
- **Output reflektiert**: Nein
- **XOR Out**: 0x00

## 5. Start- und Stoppsequenz

### 5.1 Startvorgang (Handshake)
Der Startvorgang ist eine klassische Handshake-Prozedur, um sicherzustellen, dass der Controller betriebsbereit ist, bevor das Display seine volle Funktionalität anzeigt.

1. **Button-Druck**: Der Nutzer drückt den Power-Button. Das Display wird mit Strom versorgt und beginnt sofort, Standard-TX-Pakete an den Controller zu senden. Die Display-Beleuchtung bleibt dabei zunächst aus.
2. **Controller-Antwort**: Der Motorcontroller empfängt die TX-Pakete, initialisiert sich und beginnt, RX-Pakete zu senden.
3. **Handshake-Signal**: Als Bestätigung für den erfolgreichen Start sendet der Controller mindestens ein spezielles "Boot-Acknowledged"-Paket. Dieses Paket unterscheidet sich von normalen Paketen dadurch, dass das Status-Flag an Offset 0x03 den Wert 0x80 hat.
4. **Display-Aktivierung**: Das Display wartet auf den Empfang eines gültigen RX-Pakets mit diesem Status-Flag. Sobald es dieses Paket empfängt, ist der Handshake abgeschlossen und das Display schaltet die Beleuchtung und die Benutzeroberfläche vollständig ein.
5. **Normalbetrieb**: Unmittelbar nach dem Senden des Handshake-Pakets wechselt der Controller in den normalen Betriebsmodus und sendet nur noch RX-Pakete mit einem Status-Flag von 0x00.

### 5.2 Stoppvorgang (Timeout)
Das Ausschalten erfolgt nicht über ein spezielles Kommando, sondern durch einen Kommunikationsabbruch.

1. **Button-Druck (lang)**: Der Nutzer hält den Power-Button für mehrere Sekunden gedrückt.
2. **Kommunikationsabbruch**: Das Display erkennt das lange Drücken als Ausschaltbefehl und stoppt abrupt das Senden von TX-Pakete.
3. **Controller-Timeout**: Der Motorcontroller empfängt keine Daten mehr vom Display. Nach einem internen Timeout (z.B. 200-500 ms) geht der Controller davon aus, dass das Display ausgeschaltet wurde, und versetzt sich ebenfalls in den stromsparenden Standby-Modus.

## 6. Referenztabelle der Fahrmodi

| Profil (TX 0x0C) | Fahrmodus (TX 0x04) | Max. Throttle (TX 0x10) | Min. Speed-Wert (RX 0x08) | V-Max (ca.) |
|------------------|---------------------|-------------------------|---------------------------|-------------|
| Begrenzt (0x0C19) | Stufe 1 (0x05)      | 0x0168 (360)            | ~0x00A9 (169)             | 15 km/h     |
| Begrenzt (0x0C19) | Stufe 2 (0x0A)      | 0x02A8 (680)            | ~0x0080 (128)             | 20 km/h     |
| Begrenzt (0x0C19) | Stufe 3 (0x0F)      | 0x03E8 (1000)           | ~0x0067 (103)             | 25 km/h     |
| Offen (0x0C64)    | Stufe 1 (0x05)      | 0x0168 (360)            | ~0x0086 (134)             | 19 km/h     |
| Offen (0x0C64)    | Stufe 2 (0x0A)      | 0x02A8 (680)            | ~0x0043 (67)              | 38 km/h     |
| Offen (0x0C64)    | Stufe 3 (0x0F)      | 0x03E8 (1000)           | ~0x0030 (48)              | 53 km/h     |