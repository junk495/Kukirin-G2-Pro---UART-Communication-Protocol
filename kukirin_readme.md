# Kukirin G2 Pro UART Communication Protocol
Reverse Engineering Documentation (V8)

## Introduction
This repository documents the reverse-engineered UART communication protocol between the display TFM13-FEIMI-16 (master) and the motor controller FM-G2-PRO-XS (slave) of the Kukirin G2 Pro e-scooter. All information was gathered by analyzing the data stream and validated through extensive testing.

This documentation is intended for educational purposes and for developers interested in understanding or customizing the scooter's behavior.

**Validation Status**: The protocol has been successfully validated using an Arduino Mega 2560 emulating the motor controller. The implementation passes the display's handshake validation and maintains stable communication without errors. See Section 7 for the tested reference implementation.

## 1. Physical Interface

### Basic Parameters
- **Protocol**: UART (asynchronous)
- **Baud Rate**: 9600 bps
- **Data Format**: 8N1 (8 data bits, no parity, 1 stop bit)
- **Voltage**: 5V TTL
- **Connection**: Display (Master) ↔ Motor Controller (Slave)
  - 6 pin JST SM Connector @Motor Controller
    - Red = Bat+
    - Blue = Bat+ switched
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
| 0x0C   | 1      | Calculated Status Byte | 0x6C / 0xEC           | **Critical for validation!** This value is checked by the display for consistency with the Status Flag (0x03). Calculated as: Base Value + Status Flag. Base value in normal operation is 0x6C. During handshake (Flag = 0x80), this must be 0xEC (0x6C + 0x80). An incorrect value causes error E-006. |
| 0x0D   | 1      | Unknown                | varies                | This field is not a checksum as previously assumed. Testing has shown the display does not validate this byte, and communication works without specific calculation. |
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

### 4.3 RX Packet Validation (Consistency Check)
The display does not validate incoming RX packets using a cyclic redundancy check (CRC), but rather through a simpler consistency check. The value of Byte 13 (Offset 0x0C, "Calculated Status Byte") is compared with the value of Byte 4 (Offset 0x03, "Status Flag").

**Validation Rule:**
```
Byte[0x0C] == 0x6C + Byte[0x03]
```

**Examples:**
- **Normal Operation** (Status Flag = 0x00): Calculated Status Byte must be 0x6C (108 decimal)
- **Handshake** (Status Flag = 0x80): Calculated Status Byte must be 0xEC (236 decimal = 0x6C + 0x80)

If this condition is not met, the display aborts communication and shows error code **E-006**. This validation mechanism ensures that the controller properly acknowledges the handshake and maintains packet integrity throughout operation.

## 5. Start and Stop Sequence

### 5.1 Start Procedure (Handshake)
The start procedure is a classic handshake to ensure the controller is operational before the display fully activates.

1. **Button Press**: The user presses the power button. The display is powered on and immediately starts sending standard TX packets to the controller. The display backlight remains off initially.
2. **Controller Response**: The motor controller receives the TX packets, initializes itself, and begins sending RX packets.
3. **Handshake Signal**: As confirmation of a successful start, the controller sends a specific packet sequence: The first response to a display request is a normal packet (Status Flag 0x00). Only as the **second packet** is the special "Boot-Acknowledged" packet sent with Status Flag 0x80.
4. **Consistency Check**: When sending the handshake packet (Flag 0x80), the "Calculated Status Byte" at Offset 0x0C must also be set accordingly to 0xEC (0x6C + 0x80) to pass the display's validation. Failure to do so results in error E-006.
5. **Display Activation**: The display waits for the reception of a valid RX packet with the handshake flag. Once received and validated, the handshake is complete, and the display fully activates its backlight and user interface. If the handshake validation fails, the display activates but shows error 006.
6. **Normal Operation**: Immediately after sending the handshake packet, the controller switches to normal operation mode and sends only RX packets with a status flag of 0x00 and calculated status byte of 0x6C.

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

## 7. Example Code: Arduino Mega 2560 Implementation

Below is a tested and working Arduino Mega 2560 example for emulating the motor controller. This implementation has been validated to successfully communicate with the Kukirin G2 Pro display without error codes.

**Why Arduino Mega 2560?**
The Arduino Mega 2560 is an excellent choice for this application because it features multiple hardware UART ports (Serial, Serial1, Serial2, Serial3). This enables stable and reliable serial communication without the potential timing issues associated with software-emulated serial interfaces like SoftwareSerial.

```cpp
/*
 * Kukirin G2 Pro Controller Emulator - Validated Implementation
 * 
 * Hardware: Arduino Mega 2560
 * Display TX (Green)  → Mega Pin 17 (RX2)
 * Display RX (Yellow) → Mega Pin 16 (TX2)
 * GND                 → GND
 * 
 * This code successfully passes the display's handshake validation
 * and maintains stable communication without error E-006.
 */

#include <Arduino.h>

// RX Packet Structure (16 Bytes) - Controller to Display
struct RXPacket {
    uint8_t  startMarker;      // 0x02
    uint8_t  packetLength;     // 0x0E
    uint8_t  statusType;       // 0x01
    uint8_t  statusFlag;       // 0x80 (handshake) or 0x00 (normal)
    uint8_t  systemStatus;     // 0xC0 (normal) or 0xE0 (brake)
    uint8_t  speed[3];         // Speed data (3 bytes)
    uint8_t  voltage_H;        // Voltage High Byte (Big-Endian!)
    uint8_t  voltage_L;        // Voltage Low Byte
    uint8_t  current_H;        // Current High Byte
    uint8_t  current_L;        // Current Low Byte
    uint8_t  temp_H;           // Unknown High (always 0x00)
    uint8_t  temp_L;           // Calculated Status Byte (0x6C + statusFlag)
    uint8_t  echo_H;           // Echo startMarker
    uint8_t  echo_L;           // Echo packetLength
};

// TX Packet Structure (20 Bytes) - Display to Controller
struct TXPacket {
    uint8_t  startMarker;      // 0x01
    uint8_t  packetLength;     // 0x14 (20 bytes)
    uint8_t  commandType;      // 0x01
    uint8_t  subCommand;       // 0x02
    uint8_t  staticParams1[6]; // Static configuration bytes
    uint8_t  speedProfile_L;   // Speed limit low byte
    uint8_t  speedProfile_H;   // Speed limit high byte
    uint8_t  staticParam2_L;   // Static param low
    uint8_t  staticParam2_H;   // Static param high
    uint8_t  throttle_L;       // Throttle/command low byte
    uint8_t  throttle_H;       // Throttle/command high byte
    uint8_t  checksum_L;       // Checksum low byte
    uint8_t  checksum_H;       // Checksum high byte
};

// Global Variables
RXPacket rxPacket;
TXPacket txPacket;
uint8_t rxBuffer[20];
uint8_t rxIndex = 0;
unsigned long packetCounter = 0;

// Constants
const uint8_t BASE_TEMP_VALUE = 0x6C;  // Base value for Calculated Status Byte
const uint16_t VOLTAGE_RAW = 0x0DAC;   // 35.00V (3500 = 0x0DAC)

// Forward Declarations
void initializePacket();
void handleDisplayPacket();
void sendResponse();

void setup() {
    Serial.begin(115200);   // Debug output
    Serial2.begin(9600);    // Display Communication
    
    Serial.println("===============================================");
    Serial.println("Kukirin G2 Pro Controller Emulator");
    Serial.println("===============================================");
    Serial.println("Hardware: Arduino Mega 2560");
    Serial.println("Display TX (Green) -> Pin 17 (RX2)");
    Serial.println("Display RX (Yellow) -> Pin 16 (TX2)");
    Serial.println("===============================================");
    Serial.println("Waiting for display communication...\n");
    
    // Initialize response packet
    initializePacket();
}

void loop() {
    // Receive TX packets from display
    while (Serial2.available()) {
        uint8_t inByte = Serial2.read();
        
        if (rxIndex == 0 && inByte != 0x01) {
            continue;  // Wait for start marker
        }
        
        rxBuffer[rxIndex++] = inByte;
        
        if (rxIndex >= 20) {
            handleDisplayPacket();
            rxIndex = 0;
        }
    }
}

void initializePacket() {
    rxPacket.startMarker = 0x02;
    rxPacket.packetLength = 0x0E;
    rxPacket.statusType = 0x01;
    rxPacket.statusFlag = 0x00;      // Start normal
    rxPacket.systemStatus = 0xC0;
    rxPacket.speed[0] = 0x00;
    rxPacket.speed[1] = 0x00;
    rxPacket.speed[2] = 0x00;
    rxPacket.voltage_H = (VOLTAGE_RAW >> 8) & 0xFF;  // High Byte
    rxPacket.voltage_L = VOLTAGE_RAW & 0xFF;         // Low Byte
    rxPacket.current_H = 0x00;
    rxPacket.current_L = 0x00;
    rxPacket.temp_H = 0x00;
    rxPacket.temp_L = BASE_TEMP_VALUE;  // Initial without handshake
    rxPacket.echo_H = 0x02;
    rxPacket.echo_L = 0x0E;
}

void handleDisplayPacket() {
    // Parse received packet
    memcpy(&txPacket, rxBuffer, sizeof(TXPacket));
    
    // Debug output
    Serial.print("RX #");
    Serial.print(packetCounter);
    Serial.print(": ");
    for (uint8_t i = 0; i < 20; i++) {
        if (rxBuffer[i] < 0x10) Serial.print("0");
        Serial.print(rxBuffer[i], HEX);
        Serial.print(" ");
    }
    
    // Show throttle value
    uint16_t throttle = (txPacket.throttle_H << 8) | txPacket.throttle_L;
    Serial.print(" | Throttle: 0x");
    if (throttle < 0x1000) Serial.print("0");
    if (throttle < 0x100) Serial.print("0");
    if (throttle < 0x10) Serial.print("0");
    Serial.print(throttle, HEX);
    Serial.println();
    
    packetCounter++;
    
    // Send response immediately
    sendResponse();
}

void sendResponse() {
    // CRITICAL: Handshake only on packet #2!
    if (packetCounter == 2) {
        rxPacket.statusFlag = 0x80;  // HANDSHAKE
        rxPacket.temp_L = BASE_TEMP_VALUE + 0x80;  // 0x6C + 0x80 = 0xEC
    } else {
        rxPacket.statusFlag = 0x00;  // NORMAL
        rxPacket.temp_L = BASE_TEMP_VALUE;  // 0x6C
    }
    
    // Send packet
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

/*
 * CRITICAL FINDINGS:
 * 
 * 1. HANDSHAKE SEQUENCE:
 *    - Packet 1: Status Flag = 0x00 (Normal)
 *    - Packet 2: Status Flag = 0x80 (HANDSHAKE)
 *    - Packet 3+: Status Flag = 0x00 (Normal)
 * 
 * 2. CALCULATED STATUS BYTE (0x0C):
 *    This is NOT a temperature value but a validation field!
 *    Formula: temp_L = BASE_VALUE + statusFlag
 *    
 *    Normal:    0x6C + 0x00 = 0x6C (108 decimal)
 *    Handshake: 0x6C + 0x80 = 0xEC (236 decimal)
 *    
 *    The display validates this consistency!
 *    Wrong value -> Error E-006
 * 
 * 3. NO CRC-8:
 *    Byte 0x0D is NOT a CRC-8 checksum. The display does not
 *    validate this byte. Communication works without calculation.
 */
```

### 7.1 Expected Serial Monitor Output

When running the code above, you should see output similar to this:

```
===============================================
Kukirin G2 Pro Controller Emulator
===============================================
Hardware: Arduino Mega 2560
Display TX (Green) -> Pin 17 (RX2)
Display RX (Yellow) -> Pin 16 (TX2)
===============================================
Waiting for display communication...

RX #0: 01 14 01 02 05 80 1E 00 5A 03 13 00 19 0C 01 A4 00 00 05 72  | Throttle: 0x0000
TX #1: 02 0E 01 00 C0 00 00 00 0D AC 00 00 00 6C 02 0E  | Flag: 0x00 | Calc.Status: 0x6C

RX #1: 01 14 01 02 05 80 1E 00 5A 03 13 00 19 0C 01 A4 00 00 05 72  | Throttle: 0x0000
TX #2: 02 0E 01 80 C0 00 00 00 0D AC 00 00 00 EC 02 0E  | Flag: 0x80 | Calc.Status: 0xEC <-- HANDSHAKE!

RX #2: 01 14 01 02 05 80 1E 00 5A 03 13 00 19 0C 01 A4 00 00 05 72  | Throttle: 0x0000
TX #3: 02 0E 01 00 C0 00 00 00 0D AC 00 00 00 6C 02 0E  | Flag: 0x00 | Calc.Status: 0x6C
...
```

The display should now show no error codes and display the voltage (35.0V) correctly.

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