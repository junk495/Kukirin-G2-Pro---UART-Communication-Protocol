# Kukirin G2 Pro UART Communication Protocol
Reverse Engineering Documentation (V12)

## Introduction
This repository documents the reverse-engineered UART communication protocol between the display TFM13-FEIMI-16 (master) and the motor controller FM-G2-PRO-XS (slave) of the Kukirin G2 Pro e-scooter. All information was gathered by analyzing the data stream and validated through extensive testing.

This documentation is intended for educational purposes and for developers interested in understanding or customizing the scooter's behavior.

**Validation Status**: The protocol has been successfully validated using an Arduino Mega 2560 emulating the motor controller. The implementation passes the display's handshake validation and maintains stable communication without error E-006. See Section 7 for the tested reference implementation.

**Note**: Some aspects of the protocol (e.g., speed calculation constant, unknown fields) still require physical validation with an actual Kukirin G2 Pro scooter.

## 1. Physical Interface

### Basic Parameters
- **Protocol**: UART (asynchronous)
- **Baud Rate**: 9600 bps
- **Data Format**: 8N1 (8 data bits, no parity, 1 stop bit)
- **Voltage**: 5V TTL
- **Connection**: Display (Master) ↔ Motor Controller (Slave)
  - **6-pin JST SM Connector @Motor Controller**:
    - Red = Bat+
    - Blue = Bat+ switched
    - Black = GND
    - Green = TX (Display → Controller, D0)
    - Yellow = RX (Controller → Display, D1)
- **Wheel Diameter**: 9 inches (22.86 cm, circumference ~71.81 cm)
- **Pole Count**: 30 (likely 15 pole pairs, typical for BLDC motor)
- **Speed Unit**: km/h (configurable in the display menu, switchable to mph, which may affect speed calculations)

## 2. Communication Structure

### 2.1 Timing Characteristics
- **Cycle Interval**: ~60ms between communication bursts
- **Burst Duration**: 10–15ms per complete sequence
- **Repetition Rate**: ~16–17 Hz at idle

### 2.2 Sequence Pattern
```
[TX Burst: 3 Packets] → [Pause: 2–5ms] → [RX Burst: 3 Packets] → [Idle: ~45ms]
```

### 2.3 Burst Structure
Each communication cycle consists of:
1. **TX Phase**: Display sends 3 consecutive packets (possibly for redundancy or different commands).
2. **Processing Time**: 2–5ms pause.
3. **RX Phase**: Controller responds with 3 packets (status data redundancy).
4. **Idle Phase**: ~45ms until the next cycle.

## 3. Packet Format

### 3.1 TX Packets (Display → Controller)
**Packet Length**: 20 Bytes

| Offset | Length | Designation            | Example Values        | Description & Insights                                                                 |
|--------|--------|------------------------|-----------------------|---------------------------------------------------------------------------------------|
| 0x00   | 1      | Start Marker           | 0x01                  | Fixed identifier for TX packets.                                                      |
| 0x01   | 1      | Packet Length          | 0x14                  | Always 20 bytes.                                                                      |
| 0x02   | 1      | Command Type           | 0x01                  | Status/Command.                                                                       |
| 0x03   | 1      | Sub-Command            | 0x02                  | Sub-command/Mode.                                                                     |
| 0x04   | 1      | Drive Mode             | 0x05/0x0A/0x0F        | Selects power level: 0x05 (Level 1), 0x0A (Level 2), 0x0F (Level 3).                   |
| 0x05   | 1      | Function Bitmask       | 0x80 (Base)           | Controls additional functions: +0x40 (Brake), +0x20 (Light), +0x10 (Horn), +0x08 (Blinker L), +0x04 (Blinker R). Examples: 0x80 (all off), 0xA0 (Light on), 0xC0 (Brake), 0xE0 (Light + Brake). |
| 0x06   | 2      | Parameter 2            | 0x001E                | Possibly speed limit (30 km/h, 19 decimal).                                            |
| 0x08   | 2      | Parameter 3            | 0x035A                | Unknown (858 decimal).                                                                |
| 0x0A   | 2      | Parameter 4            | 0x0013                | Unknown (19 decimal).                                                                 |
| 0x0C   | 2      | Speed Profile          | 0x0C19/0x0C64         | Selects V-Max limit: 0x0C19 (3097, Limited), 0x0C64 (3172, Open). Little-Endian.       |
| 0x0E   | 2      | Parameter 6            | 0x01A4                | Unknown (420 decimal).                                                                |
| 0x10   | 2      | Throttle Setpoint      | 0x0000–0x03E8         | Throttle position (Big-Endian). Max values: Level 1 (360), Level 2 (680), Level 3 (1000). |
| 0x12   | 2      | Indicator/Controls     | 0x0000 / 0x25XX       | Byte 0x12: Brake (0x25), Blinker L (Bit 3, 0x08), Blinker R (Bit 4, 0x10), Horn (Bit 7, 0x80). Byte 0x13: Unknown (possibly mirror/validation). |
| 0x14   | 2      | Checksum               | varies                | CRC-16/MODBUS of the first 18 bytes (Little-Endian). See Section 4.2.                  |

### 3.2 RX Packets (Controller → Display)
**Packet Length**: 16 Bytes

| Offset | Length | Designation            | Example Value         | Description & Insights                                                                 |
|--------|--------|------------------------|-----------------------|---------------------------------------------------------------------------------------|
| 0x00   | 1      | Start Marker           | 0x02                  | Response identifier.                                                                  |
| 0x01   | 1      | Packet Length          | 0x0E                  | 14 bytes of payload.                                                                  |
| 0x02   | 1      | Status Type            | 0x01                  | Response type.                                                                        |
| 0x03   | 1      | Status Flag            | 0x00 / 0x80           | 0x00 (normal), 0x80 (handshake, sent as second packet at startup).                     |
| 0x04   | 1      | System Status          | 0xC0 / 0xE0           | 0xC0 (normal), 0xE0 (brake active, Bit 5 set).                                        |
| 0x05   | 3      | Speed                  | 0x000000              | Unknown, constant at 0x000000 at idle. Possibly speed-related data.                    |
| 0x08   | 2      | Speed Raw              | 0x0DAC (Idle)         | Actual speed raw value (Big-Endian, inversely proportional to wheel speed).            |
| 0x0A   | 2      | Current                | 0x0000                | Unknown, possibly current or power (0A at idle). Not displayed on screen.              |
| 0x0C   | 1      | Calculated Status Byte | 0x6C / 0xEC           | **Critical for validation!** Calculated as: 0x6C + Status Flag (0x03). Normal: 0x6C (108 decimal). Handshake: 0xEC (236 decimal). Incorrect value causes error E-006. |
| 0x0D   | 1      | Unknown                | varies                | Not validated by display, not a CRC-8 checksum. Communication works without calculation. |
| 0x0E   | 2      | Echo/Verification      | 0x020E                | Repetition of packet header (Start Marker & Packet Length).                            |

**Note on Display Measurements**: Battery voltage and temperature are measured directly by the display’s sensors, not transmitted via UART.

## 4. Protocol Analysis & Control Logic

### 4.1 Master-Slave Principle
- The display is the master and initiates all communication.
- The controller (slave) only responds to requests.
- Fixed time slots are used for communication, ensuring a consistent cycle.

### 4.2 Checksum Calculation (TX)
The 2-byte checksum is calculated over the first 18 bytes of the packet (Offset 0x00 to 0x11) and appended in Little-Endian format.

- **Algorithm**: CRC-16/MODBUS
- **Polynomial**: 0x8005
- **Initial Value**: 0xFFFF
- **Input Reflected**: Yes
- **Output Reflected**: Yes
- **XOR Out**: 0x0000

### 4.3 RX Packet Validation (Consistency Check)
The display validates RX packets through a consistency check, not a CRC. The Calculated Status Byte (Offset 0x0C) must equal the Status Flag (Offset 0x03) plus 0x6C.

**Validation Rule:**
```
Byte[0x0C] == 0x6C + Byte[0x03]
```

- **Normal Operation**: Status Flag = 0x00 → Calculated Status Byte = 0x6C (108 decimal).
- **Handshake**: Status Flag = 0x80 → Calculated Status Byte = 0xEC (236 decimal).

Failure to meet this condition results in error **E-006**.

### 4.4 Brake Detection (Dual Encoding)
The brake signal is encoded in two ways:
- **Method 1 - Function Bitmask (Byte 5, Offset 0x05)**: Bit 6 set (value ≥ 0xC0).
- **Method 2 - Indicator Byte (Byte 18, Offset 0x12)**: Value 0x25 (primary method).
- **Controller Response**: When brake is detected, System Status (Offset 0x04) = 0xE0; otherwise, 0xC0.

### 4.5 Redundancy
- **TX Packets**: 3 packets per burst, possibly for redundancy or different commands.
- **RX Packets**: 3 packets per burst, providing status data redundancy.
- **Echo Bytes**: RX packet bytes 0x0E–0x0F echo the header (0x02, 0x0E) for error checking.

## 5. Start and Stop Sequence

### 5.1 Start Procedure (Handshake)
The handshake ensures the controller is operational before the display fully activates.

1. **Button Press**: User presses the power button. The display powers on and sends standard TX packets. The backlight remains off initially.
2. **Controller Response**: The controller initializes and responds with RX packets.
3. **Handshake Signal**: The controller sends:
   - First packet: Normal (Status Flag = 0x00, Calculated Status Byte = 0x6C).
   - Second packet: Handshake (Status Flag = 0x80, Calculated Status Byte = 0xEC).
   - Subsequent packets: Normal (Status Flag = 0x00, Calculated Status Byte = 0x6C).
4. **Display Activation**: The display validates the handshake packet (Flag = 0x80, Calculated Status Byte = 0xEC). If valid, the backlight and UI activate. If invalid, the display shows error E-006.
5. **Periodic Handshake**: Every 50 packets, the controller sends:
   - Packet % 50 == 1: Status Flag = 0x00, System Status = 0xC0, Calculated Status Byte = 0x6C.
   - Packet % 50 == 2: Status Flag = 0x80, System Status = 0xC0, Calculated Status Byte = 0xEC.
   - Other packets: Status Flag = 0x00, System Status = 0xC0 (normal) or 0xE0 (brake), Calculated Status Byte = 0x6C.

### 5.2 Stop Procedure (Timeout)
1. **Button Press (Long)**: User holds the power button for several seconds.
2. **Communication Interruption**: The display stops sending TX packets.
3. **Controller Timeout**: After 200–500ms without data, the controller enters power-saving standby mode.

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
The speed raw value (RX 0x08, 2 bytes, Big-Endian) is inversely proportional to wheel speed, likely representing scaled time per pulse (e.g., microseconds) from the motor’s Hall sensors (30 poles, 15 pole pairs, 15 pulses per revolution).

**Wheel Parameters**:
- **Wheel Diameter**: 9 inches (22.86 cm).
- **Circumference**: C = π * 22.86 ≈ 71.81 cm = 0.7181 m.
- **Pole Pairs**: 15, producing 15 pulses per wheel revolution.
- **Speed Unit**: Configured to km/h (switchable to mph, which may affect scaling).

**Speed Calculation Formula**:
```
v ≈ 2550 / Speed Raw Value
```
Where:
- v: Speed in km/h.
- Speed Raw Value: Decimal value of 2-byte field at Offset 0x08 (Big-Endian).

**Example Calculations**:
- Idle: 0x0DAC (3500 decimal) → v ≈ 2550 / 3500 ≈ 0.73 km/h ≈ 0 km/h.
- Open Profile, Level 3: 0x0030 (48 decimal) → v ≈ 2550 / 48 ≈ 53.13 km/h.

**Notes**:
- The constant 2550 is an empirical approximation (Limited: ~2535–2575, Open: ~2544–2546) due to controller calibrations.
- The raw value likely represents time per pulse, requiring physical testing to refine the constant and account for pulse frequency or gear ratios.
- Switching to mph may scale the speed raw value or V-Max, pending validation.
- Idle raw value (0x0DAC = 3500) indicates a default state, decreasing with increasing speed.

## 7. Example Code: Arduino Mega 2560 Implementation

Below is a tested Arduino Mega 2560 implementation for emulating the motor controller, validated to communicate with the display without error E-006. The Arduino Mega 2560 is ideal due to its multiple hardware UART ports (Serial, Serial1, Serial2, Serial3), ensuring reliable communication.

```cpp
/*
 * Kukirin G2 Pro Controller Emulator - Validated Implementation
 * Hardware: Arduino Mega 2560
 * Display TX (Green) → Mega Pin 17 (RX2)
 * Display RX (Yellow) → Mega Pin 16 (TX2)
 * GND → GND
 * Status: FULLY FUNCTIONAL - NO E-006 ERROR
 */

#include <Arduino.h>

// Constants
const uint16_t HANDSHAKE_INTERVAL = 50;
const uint8_t BASE_TEMP_VALUE = 0x6C;  // Base value for Calculated Status Byte
const uint16_t SPEED_RAW_IDLE = 0x0DAC;  // 3500 decimal
const uint8_t BRAKE_INDICATOR_VALUE = 0x25;

// RX Packet Structure (16 Bytes)
struct RXPacket {
    uint8_t startMarker;      // 0x02
    uint8_t packetLength;     // 0x0E
    uint8_t statusType;       // 0x01
    uint8_t statusFlag;       // 0x80 (handshake) or 0x00
    uint8_t systemStatus;     // 0xC0 (normal) or 0xE0 (brake)
    uint8_t speed[3];         // Speed data
    uint8_t speedRaw_H;       // Speed Raw High
    uint8_t speedRaw_L;       // Speed Raw Low (Big-Endian)
    uint8_t current_H;        // Current High
    uint8_t current_L;        // Current Low
    uint8_t unknown_H;        // Always 0x00
    uint8_t temp_L;           // 0x6C + statusFlag
    uint8_t echo_H;           // Echo 0x02
    uint8_t echo_L;           // Echo 0x0E
};

// TX Packet Structure (20 Bytes)
struct __attribute__((packed)) TXPacket {
    uint8_t startMarker;      // 0x01
    uint8_t packetLength;     // 0x14
    uint8_t commandType;      // 0x01
    uint8_t subCommand;       // 0x02
    uint8_t driveMode;        // 0x05/0x0A/0x0F
    uint8_t functionBitmask;  // 0x80 + bits
    uint8_t staticParam1;
    uint8_t staticParam2;
    uint8_t staticParam3;
    uint8_t staticParam4;
    uint8_t staticParam5;
    uint8_t staticParam6;
    uint8_t speedProfile_L;
    uint8_t speedProfile_H;
    uint8_t staticParam7_L;
    uint8_t staticParam7_H;
    uint8_t throttle_L;       // Big-Endian
    uint8_t throttle_H;
    uint8_t indicator_L;      // Brake/Blinker/Horn
    uint8_t indicator_H;
};

// Global Variables
RXPacket rxPacket;
TXPacket txPacket;
uint8_t rxBuffer[20];
uint8_t rxIndex = 0;
unsigned long packetCounter = 0;

void initializePacket() {
    rxPacket.startMarker = 0x02;
    rxPacket.packetLength = 0x0E;
    rxPacket.statusType = 0x01;
    rxPacket.statusFlag = 0x00;
    rxPacket.systemStatus = 0xC0;
    rxPacket.speed[0] = 0x00;
    rxPacket.speed[1] = 0x00;
    rxPacket.speed[2] = 0x00;
    rxPacket.speedRaw_H = (SPEED_RAW_IDLE >> 8) & 0xFF;
    rxPacket.speedRaw_L = SPEED_RAW_IDLE & 0xFF;
    rxPacket.current_H = 0x00;
    rxPacket.current_L = 0x00;
    rxPacket.unknown_H = 0x00;
    rxPacket.temp_L = BASE_TEMP_VALUE;
    rxPacket.echo_H = 0x02;
    rxPacket.echo_L = 0x0E;
}

void sendResponse() {
    bool brake = (txPacket.indicator_L == BRAKE_INDICATOR_VALUE);
    uint16_t cyclePosition = packetCounter % HANDSHAKE_INTERVAL;

    if (cyclePosition == 1) {
        rxPacket.statusFlag = 0x00;
        rxPacket.systemStatus = 0xC0;
        rxPacket.temp_L = BASE_TEMP_VALUE;
    } else if (cyclePosition == 2) {
        rxPacket.statusFlag = 0x80;  // HANDSHAKE
        rxPacket.systemStatus = 0xC0;
        rxPacket.temp_L = BASE_TEMP_VALUE + 0x80;  // 0xEC
    } else {
        rxPacket.statusFlag = 0x00;
        rxPacket.systemStatus = brake ? 0xE0 : 0xC0;
        rxPacket.temp_L = BASE_TEMP_VALUE;
    }

    Serial2.write((uint8_t*)&rxPacket, sizeof(rxPacket));

    // Debug output
    Serial.print("TX #");
    Serial.print(packetCounter);
    Serial.print(": ");
    uint8_t* ptr = (uint8_t*)&rxPacket;
    for (size_t i = 0; i < sizeof(rxPacket); i++) {
        if (ptr[i] < 0x10) Serial.print("0");
        Serial.print(ptr[i], HEX);
        Serial.print(" ");
    }
    Serial.print(" | Flag: 0x");
    if (rxPacket.statusFlag < 0x10) Serial.print("0");
    Serial.print(rxPacket.statusFlag, HEX);
    Serial.print(" | Calc.Status: 0x");
    if (rxPacket.temp_L < 0x10) Serial.print("0");
    Serial.print(rxPacket.temp_L, HEX);
    if (rxPacket.statusFlag == 0x80) {
        Serial.println(" <-- HANDSHAKE!");
    } else {
        Serial.println();
    }
}

void handleDisplayPacket() {
    memcpy(&txPacket, rxBuffer, sizeof(TXPacket));
    packetCounter++;

    // Debug output
    Serial.print("RX #");
    Serial.print(packetCounter);
    Serial.print(": ");
    for (uint8_t i = 0; i < 20; i++) {
        if (rxBuffer[i] < 0x10) Serial.print("0");
        Serial.print(rxBuffer[i], HEX);
        Serial.print(" ");
    }
    uint16_t throttle = (txPacket.throttle_H << 8) | txPacket.throttle_L;
    Serial.print(" | Throttle: 0x");
    if (throttle < 0x1000) Serial.print("0");
    if (throttle < 0x100) Serial.print("0");
    if (throttle < 0x10) Serial.print("0");
    Serial.print(throttle, HEX);
    Serial.println();

    sendResponse();
}

void setup() {
    Serial.begin(115200);  // Debug output
    Serial2.begin(9600);   // Display Communication
    Serial.println("===============================================");
    Serial.println("Kukirin G2 Pro Controller Emulator");
    Serial.println("===============================================");
    Serial.println("Hardware: Arduino Mega 2560");
    Serial.println("Display TX (Green) -> Pin 17 (RX2)");
    Serial.println("Display RX (Yellow) -> Pin 16 (TX2)");
    Serial.println("===============================================");
    Serial.println("Waiting for display communication...\n");

    initializePacket();
}

void loop() {
    while (Serial2.available()) {
        uint8_t inByte = Serial2.read();
        if (rxIndex == 0 && inByte != 0x01) continue;
        rxBuffer[rxIndex++] = inByte;
        if (rxIndex >= 20) {
            handleDisplayPacket();
            rxIndex = 0;
        }
    }
}
```

### 7.1 Expected Serial Monitor Output
```
===============================================
Kukirin G2 Pro Controller Emulator
===============================================
Hardware: Arduino Mega 2560
Display TX (Green) -> Pin 17 (RX2)
Display RX (Yellow) -> Pin 16 (TX2)
===============================================
Waiting for display communication...

RX #1: 01 14 01 02 05 80 1E 00 5A 03 13 00 19 0C 01 A4 00 00 05 72  | Throttle: 0x0000
TX #1: 02 0E 01 00 C0 00 00 00 0D AC 00 00 00 6C 02 0E  | Flag: 0x00 | Calc.Status: 0x6C

RX #2: 01 14 01 02 05 80 1E 00 5A 03 13 00 19 0C 01 A4 00 00 05 72  | Throttle: 0x0000
TX #2: 02 0E 01 80 C0 00 00 00 0D AC 00 00 00 EC 02 0E  | Flag: 0x80 | Calc.Status: 0xEC <-- HANDSHAKE!

RX #3: 01 14 01 02 05 80 1E 00 5A 03 13 00 19 0C 01 A4 00 00 05 72  | Throttle: 0x0000
TX #3: 02 0E 01 00 C0 00 00 00 0D AC 00 00 00 6C 02 0E  | Flag: 0x00 | Calc.Status: 0x6C
...
```

The display shows no error codes and displays speed data from UART, with battery voltage/temperature measured directly by its sensors.

## 8. Future Work
- **Physical Validation**: Test with an actual Kukirin G2 Pro scooter to confirm speed calculation constant and validate packet fields.
- **Unknown Fields**: Analyze RX fields (0x05, 0x0A, 0x0D) and TX fields (0x06–0x0B, 0x0E, 0x13) via additional data or testing.
- **Display Menu Options**: Investigate effects of settings (e.g., recuperation strength, mph vs. km/h) on the protocol.
- **Error Handling**: Document controller responses to invalid packets (e.g., incorrect checksums or lengths).
- **Firmware Updates**: Explore the firmware update protocol.
- **Boot Sequence**: Clarify full initialization process.
- **Hardware Details**: Provide motor type, UART hardware, or pin assignments for integration.

## 9. Safety Warnings
⚠️ **WARNING**: Modifications to the e-scooter can:
- Invalidate road registration.
- Void warranty.
- Lead to accidents.
- Have legal consequences.

This documentation is for educational purposes only.

## 10. Contributing
Contributions are welcome to improve this documentation. Submit pull requests with test data, results, or clarifications. Share packet captures or logs from physical tests with a Kukirin G2 Pro scooter.

## 11. License
This documentation is licensed under the MIT License. See the `LICENSE` file for details.
