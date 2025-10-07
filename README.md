# Kukirin G2 Pro UART Communication Protocol
Reverse Engineering Documentation (V8)

## Introduction
This repository documents the reverse-engineered UART communication protocol between the display TFM13-FEIMI-16 (master) and the motor controller FM-G2-PRO-XS (slave) of the Kukirin G2 Pro e-scooter. All information was gathered by analyzing the data stream and validated through various tests.

This documentation is intended for educational purposes and for developers interested in understanding or customizing the scooter's behavior.

**Note**: This is an ongoing project, and the findings presented here have not yet been tested with an actual scooter. The information reflects the current state of analysis.

## 1. Physical Interface

### Basic Parameters
- **Protocol**: UART (asynchronous)
- **Baud Rate**: 9600 bps
- **Data Format**: 8N1 (8 data bits, no parity, 1 stop bit)
- **Voltage**: 5V TTL
- **Connection**: Display (Master) ↔ Motor Controller (Slave)
  - 6 pin JST SM Connector @Motor Controller
    - Red = Bat+
    - Blue = Bat switched
    - Black = GND
    - Green = TX
    - Yellow = RX
- **Wheel Diameter**: 9 inches (22.86 cm, circumference ~71.81 cm)
- **Pole Count**: 30 (likely 15 pole pairs, typical for BLDC motor)
- **Speed Unit**: km/h (configurable in the display menu, switchable to mph, which may affect speed calculations)

## 2. Communication Structure

### 2.1 Timing Characteristics
- **Cycle Interval**: ~60ms
- **Repetition Rate**: ~16-17 Hz

### 2.2 Sequence Pattern
```
[TX Burst: 3 Packets] → [Pause: 2-5ms] → [RX Burst: 3 Packets] → [Idle: ~45ms]
```

## 3. Packet Format

### 3.1 TX Packets
The analysis has revealed the function of all basic control commands and the checksum.

**Packet Length**: 20 Bytes

| Offset | Length | Designation            | Example Values        | Description & Insights                                                                 |
|--------|--------|------------------------|-----------------------|---------------------------------------------------------------------------------------|
| 0x00   | 1      | Start Marker           | 0x01                  | Fixed identifier for TX packets.                                                      |
| 0x01   | 1      | Packet Length          | 0x14                  | Always 20 bytes.                                                                      |
| 0x02   | 2      | Command Type           | 0x0102                | Static command header.                                                                |
| 0x04   | 1      | Drive Mode             | 0x05/0x0A/0x0F        | Selects the power level (1/2/3).                                                       |
| 0x05   | 1      | Function Bitmask       | 0x80 (Base)           | Controls additional functions via bits: +0x40(Brake), +0x20(Light), +0x10(Horn), +0x08(Blinker L), +0x04(Blinker R). |
| 0x06   | 6      | Static Parameters      | constant              | Unchanged configuration values.                                                        |
| 0x0C   | 2      | Speed Profile          | 0x0C19/0x0C64         | Selects the V-Max limit (Standard/Open). Little-Endian.                                |
| 0x0E   | 2      | Static Parameter       | 0x01A4                | Unchanged configuration value.                                                         |
| 0x10   | 2      | Throttle Setpoint      | 0x0000-0x03E8         | Throttle position (Little-Endian). Maximum value is scaled by the drive mode.          |
| 0x12   | 2      | Checksum               | varies                | CRC-16/MODBUS of the first 18 bytes (Little-Endian). See Section 4.2.                  |

### 3.2 RX Packets
**Packet Length**: 16 Bytes

| Offset | Length | Designation            | Example Value         | Description & Insights                                                                 |
|--------|--------|------------------------|-----------------------|---------------------------------------------------------------------------------------|
| 0x00   | 1      | Start Marker           | 0x02                  | Response identifier.                                                                  |
| 0x01   | 1      | Packet Length          | 0x0E                  | 14 bytes of payload.                                                                  |
| 0x02   | 1      | Status Type            | 0x01                  | Response type.                                                                        |
| 0x03   | 1      | Status Flag            | 0x00 / 0x80           | 0x80 is sent once at startup as "Boot-Acknowledged". Otherwise 0x00.                   |
| 0x04   | 1      | System Status          | 0xC0 / 0xE0           | Reports brake status. 0xC0 (Normal). 0xE0 when brake is active (Bit 5 is set).         |
| 0x05   | 3      | Unknown                | 0x000000              | Constant at 0x000000 so far.                                                          |
| 0x08   | 2      | Speed                  | 0x0DAC (Idle)         | Actual speed raw value. 2-byte Big-Endian, inversely proportional to wheel speed.      |
| 0x0A   | 2      | Unknown                | 0x0000                | Unknown.                                                                              |
| 0x0C   | 1      | Brake/Regen Status     | 0x6C/0x4C             | Changes from 0x6C to 0x4C when the brake is active.                                    |
| 0x0D   | 1      | Checksum               | varies                | CRC-8 checksum of the first 13 bytes (0x00-0x0C). See Section 4.3.                     |
| 0x0E   | 2      | Echo/Verification      | 0x020E                | Repetition of the packet header (Start Marker & Packet Length).                        |

## 4. Protocol Analysis & Control Logic

### 4.1 Control Logic
- **Command Sending (TX)**: The display continuously sends packets that reflect the state of all control elements.
- **Execution (Controller)**: The controller executes the commands and limits V-Max internally based on the selected speed profile.
- **Feedback (RX)**: The controller sends status packets back, containing measurements (speed) and confirmations (brake).

### 4.2 Checksum Calculation (TX)
The 2-byte checksum is calculated over the first 18 bytes of the packet (Offset 0x00 to 0x11) and appended in Little-Endian format.

- **Algorithm**: CRC-16/MODBUS
- **Polynomial**: 0x8005
- **Initial Value**: 0xFFFF
- **Input Reflected**: Yes
- **Output Reflected**: Yes
- **XOR Out**: 0x0000

### 4.3 Checksum Calculation (RX)
The 1-byte checksum is calculated over the first 13 bytes of the packet (Offset 0x00 to 0x0C) and appended at position 14 (Offset 0x0D).

- **Algorithm**: CRC-8
- **Polynomial**: 0x31 (x⁸ + x⁵ + x⁴ + 1)
- **Initial Value**: 0x00
- **Input Reflected**: No
- **Output Reflected**: No
- **XOR Out**: 0x00

## 5. Start and Stop Sequence

### 5.1 Start Procedure (Handshake)
The start procedure is a classic handshake to ensure the controller is operational before the display fully activates.

1. **Button Press**: The user presses the power button. The display is powered on and immediately starts sending standard TX packets to the controller. The display backlight remains off initially.
2. **Controller Response**: The motor controller receives the TX packets, initializes itself, and begins sending RX packets.
3. **Handshake Signal**: As confirmation of a successful start, the controller sends at least one special "Boot-Acknowledged" packet. This packet differs from normal packets by having the status flag at Offset 0x03 set to 0x80.
4. **Display Activation**: The display waits for the reception of a valid RX packet with this status flag. Once received, the handshake is complete, and the display fully activates its backlight and user interface.
5. **Normal Operation**: Immediately after sending the handshake packet, the controller switches to normal operation mode and sends only RX packets with a status flag of 0x00.

### 5.2 Stop Procedure (Timeout)
The shutdown does not involve a specific command but occurs through a communication interruption.

1. **Button Press (Long)**: The user holds the power button for several seconds.
2. **Communication Interruption**: The display recognizes the long press as a shutdown command and abruptly stops sending TX packets.
3. **Controller Timeout**: The motor controller stops receiving data from the display. After an internal timeout (e.g., 200-500 ms), the controller assumes the display has been turned off and enters a power-saving standby mode.

## 6. Drive Mode Reference Table

| Profile (TX 0x0C) | Drive Mode (TX 0x04) | Max. Throttle (TX 0x10) | Speed Raw Value (RX 0x08) | V-Max (approx.) |
|-------------------|----------------------|-------------------------|---------------------------|-----------------|
| Limited (0x0C19)  | Level 1 (0x05)       | 0x0168 (360)            | ~0x00A9 (169)             | 15 km/h         |
| Limited (0x0C19)  | Level 2 (0x0A)       | 0x02A8 (680)            | ~0x0080 (128)             | 20 km/h         |
| Limited (0x0C19)  | Level 3 (0x0F)       | 0x03E8 (1000)           | ~0x0067 (103)             | 25 km/h         |
| Open (0x0C64)     | Level 1 (0x05)       | 0x0168 (360)            | ~0x0086 (134)             | 19 km/h         |
| Open (0x0C64)     | Level 2 (0x0A)       | 0x02A8 (680)            | ~0x0043 (67)              | 38 km/h         |
| Open (0x0C64)     | Level 3 (0x0F)       | 0x03E8 (1000)           | ~0x0030 (48)              | 53 km/h         |

### 6.1 Speed Calculation
The speed raw value (RX 0x08, 2 bytes, Big-Endian) is inversely proportional to the wheel speed, as reported by the controller. This value likely represents a scaled time per pulse (e.g., in microseconds) from the motor's Hall sensors, with 30 poles (15 pole pairs) generating 15 pulses per wheel revolution.

**Wheel Parameters**:
- **Wheel Diameter**: 9 inches (22.86 cm).
- **Circumference**: C = π * 22.86 ≈ 71.81 cm = 0.7181 m.
- **Pole Pairs**: 15, producing 15 pulses per wheel revolution.
- **Speed Unit**: Configured to km/h in the display menu. Switching to mph may scale the reported values, but this requires further analysis.

**Speed Calculation Formula**:
The approximate speed in km/h can be calculated as:
- v ≈ 2550 / Speed Raw Value
Where:
- v: Speed in km/h.
- Speed Raw Value: The decimal value of the 2-byte field at Offset 0x08 (Big-Endian).

**Example Calculations**:
- For a raw value of 0x0DAC (3500 decimal, idle state): v ≈ 2550 / 3500 ≈ 0.73 km/h ≈ 0 km/h (consistent with idle state).
- For a raw value of 0x0030 (48 decimal, Open profile, Level 3): v ≈ 2550 / 48 ≈ 53.13 km/h (matches V-Max of 53 km/h).

**Notes**:
- The constant 2550 is an empirical approximation derived from the reference table data. It may vary slightly between profiles (Limited: ~2535–2575, Open: ~2544–2546) due to internal controller calibrations.
- The raw value likely represents the time per pulse (scaled by an internal controller factor, e.g., in microseconds). Physical testing with a scooter is required to refine the constant and account for factors like exact pulse frequency or gear ratios.
- The high raw value at idle (e.g., 0x0DAC = 3500) indicates a default state, with the value decreasing as the scooter's speed increases.
- Switching the display to mph may affect the scaling of the speed raw value or the reported V-Max. This needs to be validated with physical tests.

## 7. Example Code: ESP32 Arduino Implementation

Below is an (untested) Arduino framework example for the ESP32 to control the display/controller via UART.

```cpp
#include <Arduino.h>

// UART-Konfiguration
#define RX_PIN 16
#define TX_PIN 17
#define BAUD_RATE 9600

// TX-Paket (Beispiel: Fahrmodus 1, Licht an)
uint8_t txPacket[20] = {
  0x01, 0x14, 0x01, 0x02, 0x05, 0xA0, // Start, Länge, Kommando, Fahrmodus 1, Bitmaske (Licht an)
  0x00, 0x00, 0x00, 0x00, 0x00, 0x00, // Statische Parameter
  0x0C, 0x19, // Begrenztes Speed-Profil
  0x01, 0xA4, // Statischer Parameter
  0x01, 0x68, // Throttle (Beispielwert)
  0x00, 0x00  // CRC-16 (wird berechnet)
};

// RX-Puffer
uint8_t rxPacket[16];

// CRC-Berechnungsfunktionen
uint16_t crc16_modbus(uint8_t *data, uint8_t length) {
  uint16_t crc = 0xFFFF;
  for (uint8_t i = 0; i < length; i++) {
    crc ^= data[i];
    for (uint8_t j = 0; j < 8; j++) {
      if (crc & 0x0001) {
        crc = (crc >> 1) ^ 0xA001; // Reflected polynomial
      } else {
        crc >>= 1;
      }
    }
  }
  return crc; // Little-Endian
}

uint8_t crc8(uint8_t *data, uint8_t length) {
  uint8_t crc = 0x00;
  uint8_t polynomial = 0x31;
  for (uint8_t i = 0; i < length; i++) {
    crc ^= data[i];
    for (uint8_t j = 0; j < 8; j++) {
      if (crc & 0x80) {
        crc = (crc << 1) ^ polynomial;
      } else {
        crc <<= 1;
      }
    }
    crc &= 0xFF; // Ensure 8-bit
  }
  return crc;
}

// Geschwindigkeitsberechnung
float calculateSpeed(uint16_t rawValue) {
  return 2550.0 / rawValue; // In km/h
}

void setup() {
  // Serielle Konfiguration
  Serial.begin(115200); // Für Debugging
  Serial1.begin(BAUD_RATE, SERIAL_8N1, RX_PIN, TX_PIN); // UART1 für Display/Controller

  // CRC für TX-Paket berechnen
  uint16_t crc = crc16_modbus(txPacket, 18);
  txPacket[18] = lowByte(crc);
  txPacket[19] = highByte(crc);
}

void loop() {
  // TX-Burst (3 Pakete)
  for (int i = 0; i < 3; i++) {
    Serial1.write(txPacket, 20);
    delay(2); // Pause zwischen Paketen
  }
  delay(5); // Pause nach TX-Burst

  // RX-Burst (3 Pakete empfangen)
  for (int i = 0; i < 3; i++) {
    if (Serial1.available() >= 16) {
      Serial1.readBytes(rxPacket, 16);

      // CRC überprüfen
      uint8_t calcCrc = crc8(rxPacket, 13);
      uint8_t recvCrc = rxPacket[13];
      if (calcCrc == recvCrc) {
        // Geschwindigkeit auslesen (Big-Endian)
        uint16_t speedRaw = (rxPacket[9] << 8) | rxPacket[8];
        float speedKmh = calculateSpeed(speedRaw);

        // Debugging-Ausgabe
        Serial.print("Received Packet: ");
        for (int j = 0; j < 16; j++) {
          Serial.printf("%02X ", rxPacket[j]);
        }
        Serial.println();
        Serial.print("Calculated Speed: ");
        Serial.print(speedKmh);
        Serial.println(" km/h");
      } else {
        Serial.println("CRC Error!");
      }
    }
  }

  delay(45); // Idle-Zeit
}
```

## 8. Future Work
The following areas require further investigation to enhance this documentation:
- **Physical Validation**: Test the protocol with an actual Kukirin G2 Pro scooter to confirm the speed calculation constant and validate all packet fields.
- **Unknown Fields**: Analyze the purpose of the unknown fields (RX 0x05, 0x0A) through additional data collection or testing.
- **Display Menu Options**: Investigate the impact of other display settings (e.g., recuperation strength, mph vs. km/h) on the protocol.
- **Error Handling**: Document how the controller responds to invalid packets (e.g., incorrect checksums or packet lengths).
- **Hardware Details**: Provide information on the motor type, UART hardware, or pin assignments for easier integration.
- **Contributing**: Contributions from the community, such as test data or additional reverse-engineering insights, are welcome to refine this documentation.

## 9. Contributing
We welcome contributions to improve this documentation. Please submit pull requests with additional data, test results, or clarifications. For physical tests with a Kukirin G2 Pro scooter, share packet captures or logs to help validate the findings.

## 10. License
This documentation is licensed under the MIT License. See the `LICENSE` file for details.
