# Kukirin G2 Pro UART Communication Protocol
Reverse Engineering Documentation (V14 - Acceleration & Recuperation Encoding Update)

## Introduction
This repository documents the reverse-engineered UART communication protocol between the display TFM13-FEIMI-16 (master) and the motor controller FM-G2-PRO-XS (slave) of the Kukirin G2 Pro e-scooter. All information was gathered by analyzing the data stream and validated through extensive testing.

This documentation is intended for educational purposes and for developers interested in understanding or customizing the scooter's behavior.

**Validation Status**: The protocol has been successfully validated using an Arduino Mega 2560 emulating the motor controller and physical tests with the original hardware. The implementation passes the display's handshake validation and maintains stable communication without error E-006. **NEW in V14**: Acceleration and Recuperation encoding fully validated with 8 physical tests.

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
| 0x08   | 2      | Parameter 3            | 0x035A                | Constant value 0x035A (858 decimal, Little-Endian). Unknown function. **NOT related to Recuperation/Acceleration.** |
| 0x0A   | 1      | **Recuperation + Acceleration** | 0x51 (example) | **NEW in V14**: Combined encoding of both parameters. Upper Nibble (bits 4-7): Recuperation Level (0-5). Lower Nibble (bits 0-3): Acceleration Level (1-5). Formula: `(Rekup_Level << 4) | Acc_Level`. Example: Rekup 5 + Acc 1 = 0x51. See Table 3.2. |
| 0x0B   | 1      | Parameter (Constant)   | 0x00                  | Always 0x00. Unknown function.                                                        |
| 0x0C   | 2      | Speed Profile          | 0x0C19/0x0C64         | Selects V-Max limit: 0x0C19 (3097, Limited), 0x0C64 (3172, Open). Little-Endian.       |
| 0x0E   | 2      | Parameter 6            | 0x01A4                | Unknown (420 decimal).                                                                |
| 0x10   | 2      | Throttle Setpoint      | 0x0000–0x03E8         | Throttle position (Big-Endian). Max values: Level 1 (360), Level 2 (680), Level 3 (1000). |
| 0x12   | 2      | Indicator/Controls     | 0x0005 / 0x0025       | Byte 0x12: No Brake (0x05), Brake Active (0x25, Bit 5 set +0x20). Variants: 0x0D (throttle, Bit 3 set), 0x15 (throttle, Bit 4 set). Byte 0x13: Unknown (possibly mirror/validation). |
| 0x14   | 2      | Checksum               | varies                | CRC-16/MODBUS of the first 18 bytes (Little-Endian). See Section 4.2.                  |

#### Table 3.2: Recuperation + Acceleration Combined Encoding (Byte 0x0A)

**Formula**: `Byte[0x0A] = (Recuperation_Level << 4) | Acceleration_Level`

**Validation Status**: ✅ = Physically tested and confirmed

| Rekup ↓ / Acc → | **1 (0x_1)** | **2 (0x_2)** | **3 (0x_3)** | **4 (0x_4)** | **5 (0x_5)** |
|-----------------|--------------|--------------|--------------|--------------|--------------|
| **0 (0x0_)**    | 0x01 ✅      | 0x02         | 0x03         | 0x04         | 0x05         |
| **1 (0x1_)**    | 0x11         | 0x12         | 0x13         | 0x14         | 0x15         |
| **2 (0x2_)**    | 0x21         | 0x22         | 0x23         | 0x24 ✅      | 0x25         |
| **3 (0x3_)**    | 0x31 ✅      | 0x32         | 0x33         | 0x34         | 0x35         |
| **4 (0x4_)**    | 0x41         | 0x42         | 0x43         | 0x44         | 0x45         |
| **5 (0x5_)**    | 0x51 ✅      | 0x52 ✅      | 0x53 ✅      | 0x54 ✅      | 0x55 ✅      |

**Examples**:
- Rekup 5, Acc 1: `(5 << 4) | 1 = 0x50 | 0x01 = 0x51` (binary: 0101 0001)
- Rekup 3, Acc 1: `(3 << 4) | 1 = 0x30 | 0x01 = 0x31` (binary: 0011 0001)
- Rekup 0, Acc 1: `(0 << 4) | 1 = 0x00 | 0x01 = 0x01` (binary: 0000 0001)
- Rekup 2, Acc 4: `(2 << 4) | 4 = 0x20 | 0x04 = 0x24` (binary: 0010 0100)

**Recuperation vs. Acceleration**:
- **Recuperation** (0-5): Regenerative braking strength when throttle is released. Persistent setting in display menu. Motor braking automatically activates when throttle is released.
- **Acceleration** (1-5): Throttle response/power delivery strength. Controls how aggressively the motor responds to throttle input.
- Both parameters are independent and can be set separately in the display menu.

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
| 0x0C   | 1      | Calculated Status Byte | 0x6C / 0xEC / 0x4C    | **Critical for validation!** Normally: 0x6C + Status Flag (0x03). Anomaly: 0x4C (-0x20) during brake active (System Status = 0xE0, Status Flag = 0x00). Display accepts 0x4C (no E-006). |
| 0x0D   | 1      | Unknown                | varies                | Not validated by display, not a CRC-8 checksum. Communication works without calculation. |
| 0x0E   | 2      | Echo/Verification      | 0x020E                | Repetition of packet header (Start Marker & Packet Length).                            |

**Note on Display Measurements**: Battery voltage and temperature are measured directly by the display's sensors, not transmitted via UART.

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

**Braking Anomaly**:
- During brake active (System Status = 0xE0, Status Flag = 0x00): Calculated Status Byte = 0x4C (76 decimal, -0x20 from expected 0x6C).
- **Extended Rule**: Byte[0x0C] = 0x6C + Status_Flag - (Brake_Active ? 0x20 : 0x00).
- Display accepts 0x4C during braking (no E-006 error).

Failure to meet this condition results in error **E-006**.

### 4.4 Brake Detection (Dual Encoding)
The brake signal is encoded in two ways, with the Indicator Byte as primary. Mechanical brake (lever pull) and recuperation (motor braking on throttle release) operate independently but can combine.

- **Method 1 - Function Bitmask (Byte 5, Offset 0x05)**: Bit 6 set (value ≥ 0xC0). ⚠️ Anomaly: 0xA0 (Bit 5 set) observed sporadically during throttle (light off).
- **Method 2 - Indicator Byte (Byte 18, Offset 0x12) - Primary**: No Brake (0x05), Brake Active (0x25, Bit 5 set +0x20). Variants: 0x0D/0x15 during throttle (Bits 3/4 set, unclear trigger).
- **Controller Response**: When brake detected, System Status (Offset 0x04) = 0xE0; otherwise, 0xC0. Prioritizes brake over throttle (safety feature: deceleration despite full throttle).

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
   - Other packets: Status Flag = 0x00, System Status = 0xC0 (normal) or 0xE0 (brake), Calculated Status Byte = 0x6C (or 0x4C during brake).

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
The speed raw value (RX 0x08, 2 bytes, Big-Endian) is inversely proportional to wheel speed, representing scaled time per pulse from Hall sensors (30 poles, 15 pole pairs, 15 pulses/revolution). Validated with physical tests (wheel lifted).

**Wheel Parameters**:
- **Wheel Diameter**: 9 inches (22.86 cm).
- **Circumference**: C = π * 22.86 ≈ 71.81 cm = 0.7181 m.
- **Pole Pairs**: 15, producing 15 pulses per wheel revolution.
- **Speed Unit**: Configured to km/h (switchable to mph, which may affect scaling).

**Speed Calculation Formula**:
```
v [km/h] ≈ 2550 / Speed Raw Value
```
Where:
- v: Speed in km/h.
- Speed Raw Value: Decimal value of 2-byte field at Offset 0x08 (Big-Endian).

**Example Calculations** (Validated, Level 2 Limited):
- Idle: 0x0DAC (3500 decimal) → v ≈ 2550 / 3500 ≈ 0.73 km/h.
- 7% Throttle: 0x0193 (193) → v ≈ 2550 / 193 ≈ 13.2 km/h.
- 49% Throttle: 0x0092 (146) → v ≈ 2550 / 146 ≈ 17.5 km/h.
- 100% Throttle: 0x008C (140) → v ≈ 2550 / 140 ≈ 18.2 km/h.
- 100% Throttle + Brake (decelerating): 0x08AE (2222) → v ≈ 2550 / 2222 ≈ 1.1 km/h.

**Notes**:
- Constant 2550 empirically derived and validated for Level 2 Limited (lifted wheel). On-road validation needed for all modes/profiles.
- Raw value independent of throttle (measures actual wheel rotation).
- Idle raw value (0x0DAC = 3500) decreases with increasing speed.

## 7. Example Code: Arduino Mega 2560 Implementation (v1.1)

Below is the updated Arduino Mega 2560 implementation for emulating the motor controller, now with corrected Recuperation + Acceleration decoding.
```cpp
/*
 * Kukirin G2 Pro Controller Emulator - v1.1
 * Hardware: Arduino Mega 2560
 * Display TX (Green) → Mega Pin 17 (RX2)
 * Display RX (Yellow) → Mega Pin 16 (TX2)
 * GND → GND
 * Status: FULLY FUNCTIONAL - NO E-006 ERROR
 * 
 * Version 1.1 Changes:
 * - Added proper Recuperation + Acceleration decoding (Byte 0x0A)
 * - Renamed staticParam4 to recupAccByte
 * - Added decodeRecupAcc() function
 * - Enhanced debug output
 */

#include <Arduino.h>

// Constants
const uint16_t HANDSHAKE_INTERVAL = 50;
const uint8_t BASE_TEMP_VALUE = 0x6C;
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
    uint8_t temp_L;           // 0x6C + statusFlag (or 0x4C during brake)
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
    uint8_t staticParam1;     // 0x1E
    uint8_t staticParam2;     // 0x00
    uint8_t param3_L;         // 0x5A (constant)
    uint8_t param3_H;         // 0x03 (constant)
    uint8_t recupAccByte;     // 0x0A: (Rekup<<4)|Acc
    uint8_t staticParam5;     // 0x00
    uint8_t speedProfile_L;   // 0x19/0x64
    uint8_t speedProfile_H;   // 0x0C
    uint8_t staticParam7_L;   // 0x01
    uint8_t staticParam7_H;   // 0xA4
    uint8_t throttle_H;       // Big-Endian
    uint8_t throttle_L;
    uint8_t indicator_L;      // Brake/Blinker/Horn
    uint8_t indicator_H;
};

// Recuperation + Acceleration Settings
typedef struct {
    uint8_t rekupLevel;  // 0-5
    uint8_t accLevel;    // 1-5
} RecupAccSettings;

// Global Variables
RXPacket rxPacket;
TXPacket txPacket;
uint8_t rxBuffer[20];
uint8_t rxIndex = 0;
unsigned long packetCounter = 0;

// Decode Recuperation + Acceleration from Byte 0x0A
RecupAccSettings decodeRecupAcc(uint8_t byte_0x0A) {
    RecupAccSettings settings;
    settings.rekupLevel = (byte_0x0A >> 4) & 0x0F;
    settings.accLevel = byte_0x0A & 0x0F;
    return settings;
}

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
        rxPacket.temp_L = brake ? (BASE_TEMP_VALUE - 0x20) : BASE_TEMP_VALUE;
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

    // Decode Recuperation + Acceleration
    RecupAccSettings settings = decodeRecupAcc(txPacket.recupAccByte);

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
    Serial.print(" | Throttle: ");
    Serial.print(throttle);
    
    Serial.print(" | Rekup: ");
    Serial.print(settings.rekupLevel);
    Serial.print(" | Acc: ");
    Serial.print(settings.accLevel);
    
    Serial.print(" | Brake: ");
    Serial.println(txPacket.indicator_L == BRAKE_INDICATOR_VALUE ? "ON" : "OFF");

    sendResponse();
}

void setup() {
    Serial.begin(115200);  // Debug output
    Serial2.begin(9600);   // Display Communication
    Serial.println("===============================================");
    Serial.println("Kukirin G2 Pro Controller Emulator v1.1");
    Serial.println("===============================================");
    Serial.println("Hardware: Arduino Mega 2560");
    Serial.println("Display TX (Green) -> Pin 17 (RX2)");
    Serial.println("Display RX (Yellow) -> Pin 16 (TX2)");
    Serial.println("===============================================");
    Serial.println("NEW: Recuperation + Acceleration decoding");
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
			memset(rxBuffer, 0, sizeof(rxBuffer));
        }
    }
}
```
### 7.1 Expected Serial Monitor Output
```
===============================================
Kukirin G2 Pro Controller Emulator v1.1
===============================================
Hardware: Arduino Mega 2560
Display TX (Green) -> Pin 17 (RX2)
Display RX (Yellow) -> Pin 16 (TX2)
===============================================
NEW: Recuperation + Acceleration decoding
===============================================
Waiting for display communication...

RX #1: 01 14 01 02 05 80 1E 00 5A 03 51 00 19 0C 01 A4 00 00 05 30  | Throttle: 0 | Rekup: 5 | Acc: 1 | Brake: OFF
TX #1: 02 0E 01 00 C0 00 00 00 0D AC 00 00 00 6C 02 0E  | Flag: 0x00 | Calc.Status: 0x6C

RX #2: 01 14 01 02 05 80 1E 00 5A 03 51 00 19 0C 01 A4 00 00 05 30  | Throttle: 0 | Rekup: 5 | Acc: 1 | Brake: OFF
TX #2: 02 0E 01 80 C0 00 00 00 0D AC 00 00 00 EC 02 0E  | Flag: 0x80 | Calc.Status: 0xEC <-- HANDSHAKE!

RX #3: 01 14 01 02 05 80 1E 00 5A 03 51 00 19 0C 01 A4 02 A8 25 XX  | Throttle: 680 | Rekup: 5 | Acc: 1 | Brake: ON
TX #3: 02 0E 01 00 E0 00 00 00 08 AE 00 00 00 4C 02 0E  | Flag: 0x00 | Calc.Status: 0x4C
```
The display shows no error codes and displays speed data from UART, with battery voltage/temperature measured directly by its sensors.
## 8. Changes from V13 to V14
### 8.1 Major Corrections

- Byte 0x0A Encoding: Changed from incorrect 0x50 + Acceleration_Level to correct (Rekuperation_Level << 4) | Acceleration_Level
- Table 3.1 Removed: The incorrect Rekuperation Level table from V13 has been completely removed
- Bytes 0x08-0x09: Confirmed as constant 0x5A03 (858 decimal), NOT related to Recuperation
### 8.2 New Additions

- Table 3.2: Complete 30-value matrix for all Recuperation + Acceleration combinations
- 8 Physical Tests: Validated encoding with real hardware measurements:

  - Acc 1-5 with Rekup 5 (5 tests)
  - Acc 1 with Rekup 0 (1 test)
  - Acc 1 with Rekup 3 (1 test)
  - Acc 4 with Rekup 2 (1 test)


- Arduino Code v1.1: Updated with proper decoding and enhanced debug output
### 8.3 Documentation Improvements

- Clear distinction between Recuperation and Acceleration
- Binary encoding examples for better understanding
- Formula and calculation examples added
## 9. Future Work

- On-Road Validation: Test speed formula during actual riding; compare vs. GPS; validate across all Drive Modes and Speed Profiles.
- Light Activation Test: Dedicated test with light button; monitor Function Bitmask (0x05) for Bit 5 changes; confirm 0xA0 = Light ON.
- Indicator_L Deep Dive: Monitor 0x12 during turn signal/horn activation; identify triggers for 0x0D/0x15 values.
- Calculated Status Byte Investigation: Capture longer brake sequences; analyze pattern with System Status (0x04); test with different recuperation levels.
- Complete Recuperation + Acceleration Matrix: Test remaining 22 untested combinations for full validation.
- Unknown Fields: Analyze RX fields (0x05, 0x0A, 0x0D) and TX fields (0x06-0x07, 0x0B, 0x0E-0x0F, 0x13) via additional data or testing.
- Display Menu Options: Investigate effects of settings (mph vs. km/h) on the protocol.
- Error Handling: Document controller responses to invalid packets (e.g., incorrect checksums or lengths).
- Firmware Updates: Explore the firmware update protocol.
- Boot Sequence: Clarify full initialization process.
## 10. Safety Warnings
⚠️ WARNING: Modifications to the e-scooter can:

- Invalidate road registration.
- Void warranty.
- Lead to accidents.
- Have legal consequences.
This documentation is for educational purposes only.
## 11. Contributing
Contributions are welcome to improve this documentation. Submit pull requests with test data, results, or clarifications. Share packet captures or logs from physical tests with a Kukirin G2 Pro scooter.
## 12. License
This documentation is licensed under the MIT License. See the LICENSE file for details.13. Acknowledgments

