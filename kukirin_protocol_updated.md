# Kükirin G2 Pro UART Communication Protocol - Reverse Engineering Documentation

## 1. Physical Interface

### Basic Parameters
- **Protocol**: UART (asynchronous)
- **Baud Rate**: 9600 bps
- **Data Format**: 8N1 (8 data bits, no parity, 1 stop bit)
- **Voltage**: 3.3V TTL
- **Connection**: Display (Master) ↔ Motor Controller (Slave)
- **Channels**: 
  - D0: TX (Display → Controller)
  - D1: RX (Controller → Display)

## 2. Communication Structure

### 2.1 Timing Characteristics
- **Cycle Interval**: ~60ms between communication bursts
- **Burst Duration**: 10-15ms per complete sequence
- **Repeat Rate**: ~16-17 Hz at idle

### 2.2 Sequence Pattern
```
[TX Burst: 3 Packets] → [Pause: 2-5ms] → [RX Burst: 3 Packets] → [Idle: ~45ms]
```

### 2.3 Burst Structure
Each communication cycle consists of:
1. **TX Phase**: Display sends 3 consecutive packets
2. **Processing Time**: 2-5ms pause
3. **RX Phase**: Controller responds with 3 packets
4. **Idle Phase**: ~45ms until next cycle

## 3. Packet Format

### 3.1 TX Packets (Display → Controller)
**Packet Length**: 20 Bytes

```
Offset | Length | Name              | Example Value | Description
-------|--------|-------------------|---------------|---------------------------
0x00   | 1      | Start Marker      | 0x01         | Packet identification
0x01   | 1      | Packet Length     | 0x14         | Always 20 bytes
0x02   | 1      | Command Type      | 0x01         | Status/Command
0x03   | 1      | Sub-Command       | 0x02         | Sub-command/Mode
0x04   | 1      | Drive Mode        | 0x05/0x0A/0x0F| Level 1/2/3
0x05   | 1      | Function Bitmask  | 0x80 + bits  | Light (0x20), Brake (0x40)
0x06   | 2      | Static Params 1-2 | variable     | Unknown static parameters
0x08   | 2      | Static Params 3-4 | variable     | Unknown static parameters
0x0A   | 2      | Static Params 5-6 | variable     | Unknown static parameters
0x0C   | 2      | Speed Profile     | 0x0C19/0x0C64| Limited (3097) / Open (3172)
0x0E   | 2      | Static Param 7    | variable     | Unknown
0x10   | 2      | Throttle          | 0-360/680/1000| Throttle position (BE, mode-dependent)
0x12   | 2      | Indicator/Controls| variable     | Turn signals, horn, brake (Bytes 18+19)
```

### 3.2 RX Packets (Controller → Display)
**Packet Length**: 16 Bytes

```
Offset | Length | Name              | Example Value | Description
-------|--------|-------------------|---------------|---------------------------
0x00   | 1      | Start Marker      | 0x02         | Response identifier
0x01   | 1      | Packet Length     | 0x0E         | 14 bytes payload
0x02   | 1      | Status Type       | 0x01         | Response type
0x03   | 1      | Status Flag       | 0x00         | 0=OK/Idle, 0x80=Handshake
0x04   | 1      | System Status     | 0xC0         | Battery/System (0xE0=Brake)
0x05   | 3      | Speed             | 0x000000     | 0 at idle
0x08   | 2      | Speed Raw         | 0x0DAC       | 3500 decimal at idle
0x0A   | 2      | Current           | 0x0000       | 0A at idle
0x0C   | 2      | Temperature/Sensor| 0x006C       | 108 decimal (0xEC at handshake)
0x0E   | 2      | Echo/Verification | 0x020E       | Echo of header bytes
```

## 4. Protocol Analysis

### 4.1 Master-Slave Principle
- Display is master and initiates all communication
- Controller only responds to requests
- Fixed time slots for communication

### 4.2 Redundancy
- 3 TX packets per burst (possibly redundancy or different commands)
- 3 RX packets per burst (status data redundancy)
- Echo bytes at end of RX packets for error checking

### 4.3 Drive Mode Values (Byte 4)
- `0x05` = Level 1 (lowest power)
- `0x0A` = Level 2 (medium power)
- `0x0F` = Level 3 (highest power)

### 4.4 Function Bitmask (Byte 5 - Offset 0x05)

**Base Value:** 0x80

**Additional Bits:**
- Bit 5 (0x20): Light ON
- Bit 6 (0x40): Brake (alternative encoding to Byte 18)

**Examples:**
- `0x80` = Base (everything off)
- `0xA0` = Light on (0x80 + 0x20)
- `0xC0` = Brake active (0x80 + 0x40)
- `0xE0` = Light + Brake (0x80 + 0x20 + 0x40)

### 4.5 Indicator Byte Structure (Byte 18 - Offset 0x12)

**Special Value for Brake:**
- `0x25` = Brake active (results in systemStatus 0xE0)

**Bitmask Functions:**
- Bit 3 (0x08): Turn Signal Left
- Bit 4 (0x10): Turn Signal Right  
- Bit 7 (0x80): Horn

**Byte 19 (Offset 0x13):**
- Mirror/Validation Byte (function still unclear)

**Examples:**
- `0x00 0x00` = No action
- `0x08 0x??` = Left turn signal active
- `0x10 0x??` = Right turn signal active
- `0x18 0x??` = Both signals (hazard lights)
- `0x80 0x??` = Horn active
- `0x25 0x??` = Brake active

### 4.6 Drive Mode & Speed Profile Combinations

#### Speed Profile (Bytes 12-13, Little-Endian):
- `0x0C19` (3097 decimal) = "Limited" Profile
- `0x0C64` (3172 decimal) = "Open" Profile

#### Combined Speed Limits:

**Limited Profile (0x0C19):**
- Level 1: 15 km/h max
- Level 2: 20 km/h max
- Level 3: 25 km/h max

**Open Profile (0x0C64):**
- Level 1: 19 km/h max
- Level 2: 38 km/h max
- Level 3: 53 km/h max

#### Throttle Values by Mode:
- Level 1: 0-360 (Big-Endian)
- Level 2: 0-680 (Big-Endian)
- Level 3: 0-1000 (Big-Endian)

⚠️ **Important:** Throttle values are in Big-Endian format (Byte 16 = High, Byte 17 = Low)

### 4.7 Handshake Protocol (prevents E-006 error)

The display expects periodic handshake sequences for validation:

**Timing:** Every 50 packets (configurable)

**Sequence:**
1. **Packet 1** (Position % 50 == 1):
   - statusFlag: 0x00
   - systemStatus: 0xC0
   - temp_L: 0x6C

2. **Packet 2** (Position % 50 == 2):
   - statusFlag: **0x80** ← Handshake marker!
   - systemStatus: 0xC0
   - temp_L: **0xEC** (0x6C + 0x80)

3. **Normal Packets** (all others):
   - statusFlag: 0x00
   - systemStatus: 0xC0 (normal) or 0xE0 (when braking)
   - temp_L: 0x6C

⚠️ **Without correct handshake sequence** the display shows **Error E-006**!

### 4.8 Brake Detection (Dual Encoding)

The brake can be encoded in **two ways**:

**Method 1 - Function Bitmask (Byte 5):**
- Bit 6 set: `0xC0` or higher

**Method 2 - Indicator Byte (Byte 18):**  
- Special value: `0x25`

**Controller Response when Braking:**
- systemStatus: `0xE0` (instead of 0xC0)
- temp_L: `0x6C` (unchanged)

**Important:** Byte 18 = 0x25 appears to be the primary method.

### 4.9 Interpreted RX Status Values
- **Speed Raw (Bytes 8-9)**: Big-Endian, value = raw sensor data (0x0DAC = 3500 at idle)
- **System Status (Byte 4)**: 0xC0 = normal, 0xE0 = brake active
- **Status Flag (Byte 3)**: 0x00 = normal, 0x80 = handshake
- **Temperature (Bytes 12-13)**: 0x6C = normal, 0xEC = handshake, possibly motor or controller temperature

## 5. Next Steps for Complete Decryption

### Required Measurements:
1. **While Riding**
   - Different speeds (5, 10, 15, 20, 25 km/h)
   - Acceleration and braking
   - Uphill/downhill

2. **Mode Changes**
   - ECO/Normal/Sport modes
   - Light on/off
   - Cruise control (if available)

3. **System States**
   - Different battery charge levels
   - After extended ride (warm motor)
   - Error states

### Analysis Methodology:
1. **Differential Analysis**: Compare packets in different states
2. **Checksum Reverse Engineering**: Identify checksum algorithm
3. **Bit Mask Analysis**: Which bits change with which actions

## 6. Implementation Notes

### Verified Working Implementation (Arduino C++):

```cpp
/*
 * Kükirin G2 Pro Controller Emulator - STABLE BASELINE
 * Hardware: Arduino Mega 2560
 * Display TX (Green) → Mega Pin 17 (RX2)
 * Display RX (Yellow) → Pin 16 (TX2)
 * GND → GND
 */

#include <Arduino.h>

// Configuration
const uint16_t HANDSHAKE_INTERVAL = 50;
const uint8_t BASE_TEMP_VALUE = 0x6C;
const uint16_t SPEED_RAW_IDLE = 0x0DAC;  // 3500 decimal
const uint8_t BRAKE_INDICATOR_VALUE = 0x25;

// RX Packet Structure (16 Bytes)
struct RXPacket {
    uint8_t  startMarker;      // 0x02
    uint8_t  packetLength;     // 0x0E
    uint8_t  statusType;       // 0x01
    uint8_t  statusFlag;       // 0x80 (handshake) or 0x00 (normal)
    uint8_t  systemStatus;     // 0xC0 (normal) or 0xE0 (brake)
    uint8_t  speed[3];         // Speed data (3 bytes)
    uint8_t  speedRaw_H;       // Speed Raw High Byte
    uint8_t  speedRaw_L;       // Speed Raw Low Byte (Big-Endian!)
    uint8_t  current_H;        // Current High Byte
    uint8_t  current_L;        // Current Low Byte
    uint8_t  unknown_H;        // Unknown (always 0x00)
    uint8_t  temp_L;           // Temperature/Status (0x6C + statusFlag)
    uint8_t  echo_H;           // Echo startMarker
    uint8_t  echo_L;           // Echo packetLength
};

// TX Packet Structure (20 Bytes) - received from display
struct __attribute__((packed)) TXPacket {
    uint8_t  startMarker;      // 0x01
    uint8_t  packetLength;     // 0x14
    uint8_t  commandType;      // 0x01
    uint8_t  subCommand;       // 0x02
    uint8_t  driveMode;        // 0x05/0x0A/0x0F
    uint8_t  functionBitmask;  // 0x80 base + bits
    uint8_t  staticParam1;
    uint8_t  staticParam2;
    uint8_t  staticParam3;
    uint8_t  staticParam4;
    uint8_t  staticParam5;
    uint8_t  staticParam6;
    uint8_t  speedProfile_L;   // Speed limit low byte
    uint8_t  speedProfile_H;   // Speed limit high byte
    uint8_t  staticParam7_L;
    uint8_t  staticParam7_H;
    uint8_t  throttle_L;       // Throttle low byte
    uint8_t  throttle_H;       // Throttle high byte
    uint8_t  indicator_L;      // Turn signals/Horn/Brake
    uint8_t  indicator_H;      // Mirror/Validation
};

RXPacket rxPacket;
TXPacket txPacket;
uint8_t rxBuffer[20];
uint8_t rxIndex = 0;
unsigned long packetCounter = 0;
bool lastBrakeState = false;

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
    memset((uint8_t*)&rxPacket, 0, sizeof(RXPacket));
    
    // Basic packet structure
    rxPacket.startMarker = 0x02;
    rxPacket.packetLength = 0x0E;
    rxPacket.statusType = 0x01;
    rxPacket.speed[0] = 0x00;
    rxPacket.speed[1] = 0x00;
    rxPacket.speed[2] = 0x00;
    rxPacket.speedRaw_H = (SPEED_RAW_IDLE >> 8) & 0xFF;
    rxPacket.speedRaw_L = SPEED_RAW_IDLE & 0xFF;
    rxPacket.current_H = 0x00;
    rxPacket.current_L = 0x00;
    rxPacket.unknown_H = 0x00;
    rxPacket.echo_H = 0x02;
    rxPacket.echo_L = 0x0E;
    
    uint16_t cyclePosition = packetCounter % HANDSHAKE_INTERVAL;
    
    // Handshake sequence
    if (cyclePosition == 1) {
        rxPacket.statusFlag = 0x00;
        rxPacket.systemStatus = 0xC0;
        rxPacket.temp_L = BASE_TEMP_VALUE;
    } else if (cyclePosition == 2) {
        // HANDSHAKE MARKER
        rxPacket.statusFlag = 0x80;
        rxPacket.systemStatus = 0xC0;
        rxPacket.temp_L = BASE_TEMP_VALUE + 0x80;
    } else {
        // Normal operation
        rxPacket.statusFlag = 0x00;
        bool currentBrakeState = (txPacket.indicator_L == BRAKE_INDICATOR_VALUE);
        
        if (currentBrakeState) {
            rxPacket.systemStatus = 0xE0;  // Brake active
            rxPacket.temp_L = BASE_TEMP_VALUE;
        } else {
            rxPacket.systemStatus = 0xC0;  // Normal
            rxPacket.temp_L = BASE_TEMP_VALUE;
        }
        lastBrakeState = currentBrakeState;
    }
    
    Serial2.write((uint8_t*)&rxPacket, sizeof(rxPacket));
}

void setup() {
    Serial.begin(115200);
    Serial2.begin(9600);
    initializePacket();
    Serial.println("Kukirin G2 Pro Controller Emulator - READY");
}

void loop() {
    while (Serial2.available()) {
        uint8_t inByte = Serial2.read();
        
        if (rxIndex == 0 && inByte != 0x01) {
            continue;
        }
        
        rxBuffer[rxIndex++] = inByte;
        
        if (rxIndex >= 20) {
            if (rxBuffer[0] == 0x01 && rxBuffer[1] == 0x14) {
                memcpy(&txPacket, rxBuffer, sizeof(TXPacket));
                packetCounter++;
                sendResponse();
            }
            rxIndex = 0;
            memset(rxBuffer, 0, sizeof(rxBuffer));
        }
    }
}
```

### Hardware Setup for Development:
- Logic Analyzer or Arduino/ESP32 with UART
- Optocouplers for galvanic isolation (safety!)
- Y-cable to monitor original communication

## 7. Safety Warnings

⚠️ **WARNING**: Modifications to the e-scooter can:
- Invalidate road registration
- Void warranty
- Lead to accidents
- Have legal consequences

This documentation is for educational purposes only!

## 8. Open Questions

**Clarified:**
- ✅ Handshake mechanism (every 50 packets, statusFlag 0x80)
- ✅ Brake detection (Byte 18 = 0x25 → systemStatus 0xE0)
- ✅ Turn signal encoding (Byte 18 Bits 3+4)
- ✅ Horn (Byte 18 Bit 7)
- ✅ Light (Byte 5 Bit 5)
- ✅ Drive mode values and throttle ranges
- ✅ Speed profile distinction

**Still Open:**
- ❌ Speed display encoding (Bytes 5-7 in RX packet)
- ❌ Exact checksum algorithm (TX Bytes 18-19)
- ❌ Function of Byte 19 (Indicator High)
- ❌ Meaning of static parameters (TX Bytes 6-11, 14-15)
- ❌ Unknown2 bytes in RX packet (Bytes 10-11)
- ❌ Boot sequence and initialization
- ❌ Firmware update protocol
- ❌ Other error codes besides E-006

---
*Status: October 2024 - Based on idle measurements and verified functional implementation*
*Further measurements required for complete protocol documentation*