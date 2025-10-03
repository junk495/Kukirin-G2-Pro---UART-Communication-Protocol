# Kukirin G2 Pro UART Communication Protocol - Reverse Engineering Documentation (V8)

## Introduction
This repository documents the reverse-engineered UART communication protocol between the display (master) and the motor controller (slave) of the Kukirin G2 Pro e-scooter. All information was gathered by analyzing the data stream and validated through various tests.

This documentation is intended for educational purposes and for developers interested in understanding or customizing the scooter's behavior.

**Note**: This is an ongoing project, and the findings presented here have been measured but not yet been tested with an actual scooter. 
The information reflects the current state of analysis.

## 1. Physical Interface

### Basic Parameters
- **Protocol**: UART (asynchronous)
- **Baud Rate**: 9600 bps
- **Data Format**: 8N1 (8 data bits, no parity, 1 stop bit)
- **Voltage**: 3.3V TTL
- **Connection**: Display (Master) ↔ Motor Controller (Slave)

## 2. Communication Structure

### 2.1 Timing Characteristics
- **Cycle Interval**: ~60ms
- **Repetition Rate**: ~16-17 Hz

### 2.2 Sequence Pattern
```
[TX Burst: 3 Packets] → [Pause: 2-5ms] → [RX Burst: 3 Packets] → [Idle: ~45ms]
```

## 3. Packet Format

### 3.1 TX Packets (Display → Controller) - FULLY DECODED
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

### 3.2 RX Packets (Controller → Display) - FULLY DECODED
All fields, including the checksum, have been identified.

**Packet Length**: 16 Bytes

| Offset | Length | Designation            | Example Value         | Description & Insights                                                                 |
|--------|--------|------------------------|-----------------------|---------------------------------------------------------------------------------------|
| 0x00   | 1      | Start Marker           | 0x02                  | Response identifier.                                                                  |
| 0x01   | 1      | Packet Length          | 0x0E                  | 14 bytes of payload.                                                                  |
| 0x02   | 1      | Status Type            | 0x01                  | Response type.                                                                        |
| 0x03   | 1      | Status Flag            | 0x00 / 0x80           | 0x80 is sent once at startup as "Boot-Acknowledged". Otherwise 0x00.                   |
| 0x04   | 1      | System Status          | 0xC0 / 0xE0           | Reports brake status. 0xC0 (Normal). 0xE0 when brake is active (Bit 5 is set).         |
| 0x05   | 3      | Unknown                | 0x000000              | Constant at 0x000000 so far.                                                          |
| 0x08   | 2      | Speed                  | 0x0DAC (Idle)         | Actual speed value. 2-byte Big-Endian, inversely proportional to wheel speed.          |
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

| Profile (TX 0x0C) | Drive Mode (TX 0x04) | Max. Throttle (TX 0x10) | Min. Speed Value (RX 0x08) | V-Max (approx.) |
|-------------------|----------------------|-------------------------|----------------------------|-----------------|
| Limited (0x0C19)  | Level 1 (0x05)       | 0x0168 (360)            | ~0x00A9 (169)              | 15 km/h         |
| Limited (0x0C19)  | Level 2 (0x0A)       | 0x02A8 (680)            | ~0x0080 (128)              | 20 km/h         |
| Limited (0x0C19)  | Level 3 (0x0F)       | 0x03E8 (1000)           | ~0x0067 (103)              | 25 km/h         |
| Open (0x0C64)     | Level 1 (0x05)       | 0x0168 (360)            | ~0x0086 (134)              | 19 km/h         |
| Open (0x0C64)     | Level 2 (0x0A)       | 0x02A8 (680)            | ~0x0043 (67)               | 38 km/h         |
| Open (0x0C64)     | Level 3 (0x0F)       | 0x03E8 (1000)           | ~0x0030 (48)               | 53 km/h         |
