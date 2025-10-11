# Kukirin G2 Pro - Recuperation & Brake Analysis
**Version:** v1  
**Date:** 2025-01-10  
**Status:** ✅ Validated with physical hardware testing

## 1. Introduction

This document details newly discovered protocol fields related to regenerative braking (recuperation), brake signals, and their interaction with throttle control in the Kukirin G2 Pro e-scooter. All findings are based on controlled hardware tests using the original motor controller.

**Testing Methodology:**
- Original motor controller connected (not emulated)
- Tests conducted with scooter lifted (wheel spinning freely)
- Multiple recuperation levels tested (0-5)
- Combined throttle + brake scenarios analyzed

---

## 2. Recuperation Level Encoding (TX Offset 0x0A-0x0B)

### 2.1 Discovery

The recuperation strength setting (configurable via display menu, 6 levels: 0-5) is transmitted in the TX packet at **Offset 0x0A-0x0B** (previously documented as "Parameter 4 - Unknown").

### 2.2 Encoding Formula
```
Byte[0x0A] = 0x03 + (Recuperation_Level * 0x10)
Byte[0x0B] = 0x00 (always)
```
### 2.3 Complete Value Table

| Recuperation Level | Byte 0x0A | Byte 0x0B | Decimal | Description |
|-------------------|-----------|-----------|---------|-------------|
| **0 (Off)** | `0x03` | `0x00` | 3 | No motor braking |
| **1 (Weak)** | `0x13` | `0x00` | 19 | Minimal regeneration |
| **2 (Low)** | `0x23` | `0x00` | 35 | Light motor braking |
| **3 (Medium)** | `0x33` | `0x00` | 51 | Moderate motor braking |
| **4 (High)** | `0x43` | `0x00` | 67 | Strong motor braking |
| **5 (Maximum)** | `0x53` | `0x00` | 83 | Maximum regeneration |

### 2.4 Validation Status

✅ **Fully Validated** - All six levels tested with brake-on/brake-off sequences. Formula confirmed across all levels.

### 2.5 Behavior Notes

- **Permanent Setting:** Recuperation level is a persistent configuration sent in every TX packet
- **Independent from Brake Signal:** Recuperation (motor braking when throttle released) operates independently from mechanical brake (Offset 0x12)
- **Controller Response:** Controller receives the level but does NOT echo it back in RX packets
- **Motor Control:** The controller uses this value internally to adjust motor braking strength when throttle is released

---

## 3. Brake Signal Analysis

### 3.1 Brake Encoding Confirmation

**TX Packet (Display → Controller):**
- **Offset 0x12 (Indicator_L):**
  - No Brake: `0x05`
  - Brake Active: `0x25` (Bit 5 set, +0x20)

**RX Packet (Controller → Display):**
- **Offset 0x04 (System Status):**
  - Normal: `0xC0`
  - Brake Active: `0xE0` (Bit 5 set, +0x20)

### 3.2 New Discovery: Simultaneous Throttle + Brake Operation

**Critical Finding:** The brake signal and throttle can be **simultaneously active**. The controller prioritizes braking over throttle input.

**Test Scenario:**
1. Scooter lifted, wheel spinning freely
2. Throttle applied to 100% (0x02A8 for Level 2)
3. Brake lever pulled while maintaining full throttle
4. Brake lever released while maintaining full throttle

**Results:**

| Phase | Throttle (0x10-0x11) | Indicator (0x12) | Speed Raw (RX) | System Status (RX) | Behavior |
|-------|---------------------|------------------|----------------|-------------------|----------|
| Full Throttle | `02 A8` (680) | `05` | `00 8C` (140) | `C0` | Wheel accelerating |
| **Brake Applied** | `02 A8` (680) | **`25`** | `08 AE` (2222) | **`E0`** | **Wheel decelerating despite throttle!** |
| **Brake Held** | `02 A8` (680) | `25` | `0D AC` (3500) | `E0` | **Wheel stopped** |
| Brake Released | `02 A8` (680) | `05` | `00 8C` (140) | `C0` | Wheel re-accelerating |

**Interpretation:**
- Controller **prioritizes brake signal** over throttle command
- Even with 100% throttle, brake causes immediate deceleration
- This is a **safety feature** preventing unintended acceleration while braking

### 3.3 Brake vs. Recuperation

**Key Distinction:**

| Feature | Location | Type | Trigger |
|---------|----------|------|---------|
| **Recuperation** | TX 0x0A | Permanent setting | Throttle released (automatic) |
| **Mechanical Brake** | TX 0x12 | Temporary signal | Brake lever pulled (manual) |

Both can operate **simultaneously** and **independently**:
- High recuperation (Level 5) + Brake lever pulled = Maximum braking force
- Low recuperation (Level 1) + Brake lever pulled = Brake only, minimal regen

---

## 4. Speed Raw Value Analysis (RX Offset 0x08-0x09)

### 4.1 Physical Validation

The Speed Raw value at **RX Offset 0x08-0x09** (Big-Endian) was validated using a lifted scooter test with controlled throttle and brake inputs.

### 4.2 Confirmed Behavior

**Speed Raw is:**
- ✅ **Inversely proportional** to wheel speed
- ✅ **Independent** from throttle input (measures actual wheel rotation)
- ✅ **Real-time measurement** from Hall sensors (30 poles, 15 pulses/revolution)
- ✅ Updates continuously (~60ms intervals)

### 4.3 Test Results

| Throttle % | Speed Raw (Hex) | Speed Raw (Dec) | Calculated Speed (v ≈ 2550/Raw) | Status |
|------------|-----------------|-----------------|----------------------------------|--------|
| 0% | `0D AC` | 3500 | 0.7 km/h | Idle/Stopped |
| 7% | `00 C1` | 193 | 13.2 km/h | Accelerating |
| 49% | `00 92` | 146 | 17.5 km/h | Accelerating |
| 100% | `00 8C` | 140 | 18.2 km/h | Max speed (Level 2 Limited) |
| 100% + Brake | `08 AE` | 2222 | 1.1 km/h | Decelerating |
| 100% + Brake (stopped) | `0D AC` | 3500 | 0.7 km/h | Wheel stopped |

### 4.4 Speed Calculation Formula
```
v [km/h] ≈ 2550 / Speed_Raw_Decimal
```
Where:
- `Speed_Raw_Decimal` = (Byte[0x08] << 8) | Byte[0x09] (Big-Endian)
- Constant **2550** is empirically derived (validated for Level 2, Limited profile)

**Examples:**
- Idle: 2550 / 3500 ≈ **0.73 km/h**
- Max Speed (Level 2): 2550 / 140 ≈ **18.2 km/h**

### 4.5 Validation Notes

⚠️ **Status:** Validated with lifted scooter (free wheel spin). On-road validation required to confirm:
- Constant accuracy across all Drive Modes (Level 1/2/3)
- Constant accuracy across Speed Profiles (Limited/Open)
- Behavior during actual riding conditions (wheel under load)

---

## 5. RX Packet Anomaly: Calculated Status Byte During Braking

### 5.1 Unexpected Behavior

The **Calculated Status Byte (RX Offset 0x0C)** shows an anomaly during braking that contradicts the documented validation rule.

**Original Documentation (README_V12.md) states:**
```
Byte[0x0C] == 0x6C + Byte[0x03]
```
Where:
- Normal: Status Flag (0x03) = 0x00 → Calculated Status = 0x6C
- Handshake: Status Flag (0x03) = 0x80 → Calculated Status = 0xEC

### 5.2 Observed Anomaly

**During Brake Events:**

| Condition | Status Flag (0x03) | System Status (0x04) | Calculated Status (0x0C) | Expected | Actual | Difference |
|-----------|-------------------|---------------------|-------------------------|----------|--------|------------|
| Normal | `0x00` | `0xC0` | `0x6C` | 0x6C | ✅ 0x6C | Correct |
| **Brake Active** | `0x00` | `0xE0` | **`0x4C`** | 0x6C | ❌ 0x4C | **-0x20** |

### 5.3 Analysis

**The value changes from 0x6C to 0x4C (-0x20) when braking, despite Status Flag remaining 0x00.**

**Possible Explanations:**
1. **Alternate Validation Rule:** `Byte[0x0C] = 0x6C + Status_Flag - (Brake_Active ? 0x20 : 0x00)`
2. **Controller Bug:** Unintended behavior in firmware
3. **Undocumented Feature:** Additional validation tied to System Status (0x04) rather than Status Flag (0x03)

⚠️ **Status:** Anomaly confirmed across multiple brake tests. Display accepts 0x4C as valid during braking (no E-006 error). Further investigation needed.

---

## 6. Unknown Fields Requiring Further Analysis

### 6.1 TX Offset 0x12 (Indicator_L) - Variant Values

During throttle testing, Indicator_L showed **unexpected values** beyond the documented `0x05` (normal) and `0x25` (brake):

| Value | Binary | Hex | Observed Context | Interpretation |
|-------|--------|-----|-----------------|----------------|
| `0x05` | 00000101 | 0x05 | Normal operation | Base value |
| `0x0D` | 00001101 | 0x0D | During throttle | Bit 3 set (+0x08) |
| `0x15` | 00010101 | 0x15 | During throttle | Bit 4 set (+0x10) |
| `0x25` | 00100101 | 0x25 | Brake active | Bit 5 set (+0x20) |

**Status:** ❓ Requires further testing. Values `0x0D` and `0x15` appeared sporadically during throttle operation without user input on turn signals or horn. May be related to controller state or packet timing.

### 6.2 TX Offset 0x05 (Function Bitmask) - Bit 5 Anomaly

**Observed Value:** `0xA0` (Bit 5 set, +0x20) appeared **once** during throttle testing.

**Original Documentation states:**
- Bit 6 (0x40): Brake
- Bit 5 (0x20): Light

**Problem:** Light was **NOT activated** during the test (confirmed by user).

**Possible Causes:**
1. Packet corruption or timing glitch
2. Controller-initiated signal (not user-triggered)
3. Misinterpretation of bit function (Bit 5 may not be "Light")

**Status:** ❓ Single occurrence, not reproducible. Requires dedicated light-activation test to validate Bit 5 function.

---

## 7. Summary of New Findings

### ✅ Fully Validated

1. **Recuperation Level Encoding (TX 0x0A-0x0B)**
   - Formula: `0x03 + (Level * 0x10)`
   - All 6 levels tested (0-5)
   - Independent from brake signal

2. **Simultaneous Throttle + Brake Operation**
   - Controller prioritizes brake over throttle
   - Safety feature confirmed

3. **Speed Raw Behavior (RX 0x08-0x09)**
   - Inversely proportional to wheel speed
   - Formula: `v ≈ 2550 / Speed_Raw`
   - Validated with lifted scooter test

### ⚠️ Partially Validated

1. **Calculated Status Byte Anomaly (RX 0x0C)**
   - Changes to 0x4C during braking (expected 0x6C)
   - Display accepts value (no error)
   - Mechanism unclear

### ❓ Requires Further Testing

1. **Indicator_L Variant Values (TX 0x12)**
   - Values 0x0D, 0x15 observed
   - No clear trigger identified

2. **Function Bitmask Bit 5 (TX 0x05)**
   - Single occurrence of 0xA0
   - Light function not confirmed

---

## 8. Recommendations for Future Testing

1. **On-Road Speed Validation:**
   - Test Speed Raw formula during actual riding
   - Compare calculated speed vs. GPS speed
   - Validate across all Drive Modes and Speed Profiles

2. **Light Activation Test:**
   - Dedicated test with light button pressed
   - Monitor Function Bitmask (0x05) for Bit 5 changes
   - Confirm whether 0xA0 = Light ON

3. **Indicator_L Deep Dive:**
   - Monitor 0x12 during turn signal activation
   - Monitor 0x12 during horn activation
   - Identify trigger conditions for 0x0D, 0x15 values

4. **Calculated Status Byte Investigation:**
   - Capture longer brake sequences
   - Analyze pattern relationship with System Status (0x04)
   - Test with different recuperation levels active

---

## 9. Updated TX Packet Table (Offset 0x0A-0x0B)

| Offset | Length | Designation | Example Values | Description |
|--------|--------|-------------|----------------|-------------|
| 0x0A | 1 | **Recuperation Level (Low)** | **0x03/0x13/0x23/0x33/0x43/0x53** | **Regenerative braking strength. Formula: 0x03 + (Level * 0x10). Range: 0 (Off) to 5 (Maximum).** |
| 0x0B | 1 | **Recuperation Level (High)** | **0x00** | **Always 0x00 (16-bit field, only low byte used).** |

---

## 10. License

This documentation is licensed under the MIT License. See the main repository LICENSE file for details.

---

**Contributors:** Protocol reverse engineering and validation by the Kukirin G2 Pro community.

