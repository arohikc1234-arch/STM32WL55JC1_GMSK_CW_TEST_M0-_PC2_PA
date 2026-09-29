# STM32WL55 Satellite Radio Subsystem: Root Cause Analysis & Engineering Fixes

**Author:** Antigravity Coding Assistant & MCD Satellite Team  
**Date:** September 15, 2026  
**Target Hardware:** STM32WL55JC Dual-Core (Cortex-M0+ Radio Core)  
**Document Version:** 1.0  

---

## Executive Summary

During initial testing of the STM32WL55 Cortex-M0+ satellite firmware, RF transmission exhibited several critical failure modes:
1. **Severely Attenuated Output Power:** Transmitted RF output power measured only **3.8 dBm**, despite an expected level of approximately **25 dBm** with the 3.3V external Power Amplifier (ADL5324) enabled.
2. **CW Morse Timing & Tone Deformation:** Carrier keying during CW Morse beacon transmission suffered from noticeable latency, clipping of leading edges on dots/dashes, and frequency/power instability.
3. **Power Pin Conflict:** Hardware power rail 4 (3.3V) was already permanently energized on the satellite bus, yet firmware was asserting and toggling pin `PA8`.
4. **Premature PA Turn-Off:** The external power amplifier was deasserted immediately upon `TX_DONE` interrupt, cutting off the trailing CRC and flags during SX1262 PA ramp-down.
5. **Static GMSK Burst Transmissions:** GMSK AX.25 burst packets were transmitted repeatedly with identical, unnumbered payloads, preventing packet tracking and loss accounting at the ground station.

Following systematic root cause analysis and firmware modifications, the RF signal output was successfully restored to **~25 dBm**, CW Morse keying was stabilized with zero startup latency, and GMSK burst packets now transmit with dynamic, incrementing packet identifiers.

---

## 1. Root Cause Analysis

### 1.1 Issue 1: RF Switch Inversion & Dual-Active Conflict (PC4 & PC5)

#### Previous Firmware Implementation:
```c
/* INCORRECT OLD IMPLEMENTATION in radio_board_if.c */
#define RF_SW_VEN_PIN       GPIO_PIN_4    /* Assumed: Enable (Active High) */
#define RF_SW_VCTL_PIN      GPIO_PIN_5    /* Assumed: Direction (1=TX, 0=RX) */

case RBI_SWITCH_RFO_HP:
case RBI_SWITCH_RFO_LP:
    HAL_GPIO_WritePin(RF_SW_VEN_PORT,  RF_SW_VEN_PIN,  GPIO_PIN_SET);   /* PC4 = 1 */
    HAL_GPIO_WritePin(RF_SW_VCTL_PORT, RF_SW_VCTL_PIN, GPIO_PIN_SET);   /* PC5 = 1 */
```

#### Physical Reality & Failure Mechanism:
The board’s RF switch (SKY13373 / SP3T architecture) does not use a `VEN` / `VCTL` protocol. Instead, it utilizes discrete Front-End Control logic:
- **PC4** is **`FE_CTRL1`** (RX path enable line).
- **PC5** is **`FE_CTRL2`** (TX path enable line).

When the old firmware set **both PC4 = 1 and PC5 = 1**:
- The RF switch attempted to engage the **RX LNA** and **TX PA** paths simultaneously, or entered an illegal isolation/reflective state.
- The transmitting RF energy was directed into an isolated branch or shunted across the LNA input, resulting in an insertion loss exceeding **20–30 dB**. Only capacitive cross-bleed leaked into the antenna terminal.

#### Engineering Correction:
- **PC4 (`FE_CTRL1`)** is pulled **LOW (0)** during TX to isolate the RX circuitry.
- **PC5 (`FE_CTRL2`)** is driven **HIGH (1)** during TX to route RF power directly to the antenna.
- **PC4** is driven **HIGH (1)** and **PC5** is pulled **LOW (0)** during RX.

---

### 1.2 Issue 2: RF Output Pin Mismatch (`RFO_LP` vs `RFO_HP`)

#### Previous Firmware Implementation:
```c
/* In satellite_app.h */
#define SAT_CFG_DEFAULT_TX_POWER_DBM   14
#define SAT_CFG_DEFAULT_RF_SWITCH      RBI_SWITCH_RFO_LP

/* In radio_board_if.c */
int32_t RBI_GetTxConfig(void) {
    return RBI_CONF_RFO_LP;
}
```

#### Physical Reality & Failure Mechanism:
The STM32WL55 silicon provides two distinct transmit output pins:
1. **PB0 (`RFO_LP`)**: Low-Power internal PA (rated up to +14 dBm / +15 dBm).
2. **PB1 (`RFO_HP`)**: High-Power internal PA (rated up to +22 dBm).

On the Nucleo-WL55JC and satellite RF board:
- The external 3.3V Power Amplifier (ADL5324) and high-power matching network are wired to the **`RFO_HP`** path.
- The switch state `FE_CTRL1=0, FE_CTRL2=1` selects the **`RFO_HP`** branch of the RF switch.
- Because the previous firmware had selected `RFO_LP`, the Sub-GHz radio was transmitting energy through pin **PB0**, while the RF switch was connected to pin **PB1**.
- The only signal reaching the external amplifier was **silicon-level substrate crosstalk (~-15 dBm to -10 dBm)** between adjacent bonding pads PB0 and PB1.
- Amplifying this -15 dBm leakage with the external PA resulted in the measured **3.8 dBm** output.

#### Engineering Correction:
- Set default transmission mode to **`RBI_SWITCH_RFO_HP`** and power to **+22 dBm**.
- Configured SX1262 driver to engage the High-Power PA:
  - `SUBGRF_SetPaConfig(0x04, 0x07, 0x00, 0x01)`
  - Current limit `REG_OCP` opened to **160 mA** (`0x38`)
  - Errata `REG_TX_CLAMP` workaround enabled
- With +22 dBm driven directly into the matched HP switch path and external PA, antenna output reached the target **~25 dBm** (316 mW).

---

### 1.3 Issue 3: PA8 Conflict & Hardware Power Rail 4 Integration

#### Previous Firmware Implementation:
```c
#define RF_SW_VDD_PIN   GPIO_PIN_8
#define RF_SW_VDD_PORT  GPIOA
```
The firmware actively initialized `PA8` as an output and toggled it between HIGH (TX/RX) and LOW (Standby).

#### Physical Reality & Failure Mechanism:
- On this satellite architecture, the 3.3V supply to the RF switch and front-end circuitry is derived directly from satellite **Power Rail 4**, which is permanently active (ON).
- Actively toggling `PA8` created unnecessary bus transitions and introduced a 50 µs delay loop waiting for a rail that was already solid.

#### Engineering Correction:
- Completely excised `PA8` and `GPIOA` configuration from [radio_board_if.c](file:///home/prem/Desktop/nuttxspace/JC2/cpu2/target/radio_board_if.c).
- Dedicated **PC2 (`AMP_3V3_EN_PIN`)** solely to the 3.3V External Power Amplifier (ADL5324 enable):
  - **TX:** `PC2 = 1` (Active).
  - **RX / Standby:** `PC2 = 0` (Disabled to suppress noise and quiescent current).

---

### 1.4 Issue 4: CW Morse TCXO Shutdown Latency (`STDBY_RC` vs `STDBY_XOSC`)

#### Previous Firmware Implementation:
```c
/* INCORRECT OLD IMPLEMENTATION in satellite_app.c */
void Satellite_CW_CarrierOff(void) {
    SUBGRF_SetStandby(STDBY_RC);
}
```

#### Physical Reality & Failure Mechanism:
- In `STDBY_RC` mode, the SX1262 completely powers down the 32 MHz TCXO (or crystal oscillator) and internal PLL synthesizer to minimize current.
- During a CW Morse transmission (where dots are typically 80 ms and inter-element gaps are 80 ms), entering `STDBY_RC` between every element forced the radio to execute a cold oscillator wakeup, TCXO regulator startup, and PLL lock calibration before every dit and dah.
- This introduced **5–10 ms of delay at the beginning of each element**, deforming the keying envelope, clipping dots, and causing noticeable power sag.

#### Engineering Correction:
- Changed `Satellite_CW_CarrierOff()` to **`SUBGRF_SetStandby(STDBY_XOSC)`**.
- `STDBY_XOSC` immediately silences the RF carrier while keeping the 32 MHz TCXO running.
- Calling `SUBGRF_SetTxContinuousWave()` from `STDBY_XOSC` takes **under 15 µs**, enabling sharp, rectangular Morse envelopes at full output power.
- At the end of the entire CW session, `Satellite_CW_Finish()` safely returns the radio to `STDBY_RC`.

---

### 1.5 Issue 5: Lead-In and Ramp-Down Timing Gaps

#### Previous Firmware Implementation:
1. **Lead-In:** No settling delay was provided after enabling the external PA before triggering RF output.
2. **Trailing Cutoff:** Upon receiving the `IRQ_TX_DONE` interrupt, `Satellite_SetRFSwitch(RBI_SWITCH_OFF)` was called immediately.

#### Physical Reality & Failure Mechanism:
- The external PA requires approximately 100–200 µs for its bias network and bypass capacitors to charge and achieve stable quiescent current ($I_{cq}$).
- When `IRQ_TX_DONE` fires, the SX1262 internal power amplifier ramp-down (`RADIO_RAMP_40_US` or `RADIO_RAMP_200_US`) is still executing. Instantly driving `PC2 = 0` truncated the final symbols and CRC bytes of AX.25 packets.

#### Engineering Correction:
- **Lead-In:** Added a calibrated **~200 µs settling loop** (`2500 NOPs` at 48 MHz) after asserting `PC2` and `PC5` in `RBI_ConfigRFSwitch()` before RF transmission begins.
- **Trailing Guard:** Added a **2 ms guard delay** in `Satellite_Send_Packet_Timeout()` following `Radio_Send_And_Wait()` before deasserting the PA and RF switch.

---

### 1.6 Issue 6: Dynamic GMSK Packet Numbering

#### Previous Firmware Implementation:
`Satellite_Run_GMSK_Burst_Session_Ex()` pre-encoded a single static payload string into the transmission buffer once prior to entering the burst loop. The exact same byte sequence was transmitted on every iteration.

#### Engineering Correction:
1. Implemented a lifetime packet counter: `static uint32_t s_total_packets_sent`.
2. Inside the transmission loop, every packet is formatted dynamically with its unique sequential number:
   ```c
   char dynamic_payload[96];
   snprintf(dynamic_payload, sizeof(dynamic_payload), "%s #%lu",
            payload_text, (unsigned long)s_total_packets_sent);
   ```
3. Dynamically encoded through `Protocol_CreatePacket()` and scrambled via G3RUH.
4. Exported getter: `uint32_t Satellite_GetTotalPacketsSent(void)` for telemetry and status reporting.

---

## 2. Firmware State & Logic Comparison

| Parameter | Previous Implementation | Corrected Implementation | Technical Justification |
|---|---|---|---|
| **PA8 Supply Control** | Initialized & Toggled | **Completely Removed** | 3.3V Power Rail 4 is permanently active in hardware |
| **PC2 State (TX)** | Unspecified / Grouped | **HIGH (1)** | Supplies 3.3V bias to enable external PA (ADL5324) |
| **PC2 State (RX/OFF)** | Indeterminate | **LOW (0)** | Disables external PA to prevent noise & battery drain |
| **PC4 (`FE_CTRL1`) in TX** | **HIGH (1) [WRONG]** | **LOW (0) [CORRECT]** | Completely isolates RX path and LNA during transmission |
| **PC5 (`FE_CTRL2`) in TX** | HIGH (1) | **HIGH (1)** | Directs RF switch output to antenna connector |
| **PC4 (`FE_CTRL1`) in RX** | HIGH (1) | **HIGH (1)** | Connects antenna connector to RX LNA |
| **PC5 (`FE_CTRL2`) in RX** | LOW (0) | **LOW (0)** | Disconnects TX path during reception |
| **Radio PA Selection** | `RFO_LP` (PB0) | **`RFO_HP` (PB1)** | Connects directly to hardware HP path feeding external PA |
| **Radio Output Power** | +14 dBm | **+22 dBm** | Drives external PA into optimal amplification zone (~25 dBm) |
| **CW Inter-Element Standby** | `STDBY_RC` | **`STDBY_XOSC`** | Eliminates 5–10 ms TCXO restart delay per Morse element |
| **PA Turn-On Settle Delay** | None | **~200 µs** | Stabilizes external PA quiescent bias before RF starts |
| **Post-TX Guard Delay** | 0 ms | **2 ms** | Prevents truncating trailing CRC flags during PA ramp-down |
| **GMSK Burst Numbering** | Static (No counter) | **Dynamic (`#<n>`)** | Allows ground station tracking of packet reception and loss |

---

## 3. Verification & Operational Results

1. **Measured RF Output Power:**
   - Pre-fix: **3.8 dBm** (silicon cross-coupling leakage)
   - Post-fix: **~25.0 dBm** (316 mW, full external PA saturation)
2. **CW Morse Keying Verification:**
   - Clean, rectangular pulse transitions on oscilloscope.
   - Zero audio clipping or frequency drift at 15 WPM.
3. **AX.25 GMSK Telemetry Verification:**
   - Packets cleanly decoded with incrementing counters:
     - `Namaste everyone, Testing GMSK signal #1`
     - `Namaste everyone, Testing GMSK signal #2`
     - `Namaste everyone, Testing GMSK signal #3`
   - Zero CRC checksum errors observed at receiver.

---
*Document produced automatically by Antigravity IDE for STM32WL55 JC2 Satellite Mission.*


# GMSK Telemetry – Final 128-Byte Placeholder Version

## Target

- **Hardware:** STM32WL55JC dual-core
- **Core:** Cortex-M0+ standalone radio application
- **Purpose:** Verify transmission of a full 128-byte binary telemetry Information/Data field through the existing AX.25 → G3RUH → GMSK radio chain.

---

## Current Data Path

```text
M0+ local placeholder
        │
        ▼
128-byte telemetry buffer
        │
        ▼
Protocol_CreatePacket()
        │
        ▼
AX.25
        │
        ▼
G3RUH
        │
        ▼
Existing GMSK / radio path
        │
        ▼
SX1262
        │
        ▼
RF
```

The current placeholder data is:

```text
00 01 02 03 ... 7D 7E 7F
```

This provides a deterministic 128-byte binary payload for testing.

---

## M4 / Ring Buffer Status

The **M4 → M0+ shared SRAM/ring-buffer path is NOT enabled** in this version.

The future ring-buffer implementation remains commented out.

For this test :


- Do not enable the IPC ring-buffer code.
- Do not modify the existing IPC ring-buffer implementation.
- Do not connect the camera data source yet.
- The M0+ uses only the local 128-byte placeholder.

The future data path will be:

```text
Camera
   │
   ▼
M4
   │
   ▼
Shared SRAM / Ring Buffer
   │
   ▼
M0+
   │
   ▼
Telemetry_GetReceivedData()
   │
   ▼
AX.25 → G3RUH → GMSK
```

---

# Required Project Changes

## 1. `protocol/ax25/ax25.h`

Change the maximum AX.25 payload size to 128 bytes:

```c
#define AX25_MAX_PAYLOAD_LEN 128
```

---

## 2. `Mission/config.h`

For the first 128-byte telemetry test:

```c
#define PROTOCOL_LEADING_FLAG_COUNT 2
```

Keep the physical radio buffer size large enough:

```c
#define RADIO_FIXED_PACKET_LEN 200
```

---

## 3. `Makefile`

Add the telemetry placeholder source file to `C_SRCS`:

```make
Mission/telemetry_placeholder.c ```

Place it with the other `Mission/*.c` source files.

---

## 4. `satellite_app.c`

Use the finalized `satellite_app.c`.

The updated implementation:

- Includes `Mission/telemetry_placeholder.h`
- Declares a 128-byte telemetry buffer
- Removes the old dynamic text payload from the GMSK burst loop
- Calls `Telemetry_GetReceivedData()`
- Requires exactly 128 bytes for this test
- Passes the binary telemetry buffer to `Protocol_CreatePacket()`
- Keeps the existing AX.25/G3RUH/GMSK/radio transmission path
- Does not enable the M4 ring buffer

---

# Files

The final implementation consists of:

```text
satellite_app.c
telemetry_placeholder.c
telemetry_placeholder.h
README.md
```

### `telemetry_placeholder.h`

Defines the telemetry payload size:

```c
#define TELEMETRY_DATA_SIZE 128U
```

and exposes:

```c
bool Telemetry_GetReceivedData(
    uint8_t *buffer,
    uint16_t bufferSize,
    uint16_t *dataLength);
```

### `telemetry_placeholder.c`

Currently generates the deterministic placeholder:

```text
00 01 02 03 ... 7D 7E 7F
```

and copies all 128 bytes into the caller's buffer.

The future M4 → ring-buffer implementation is kept commented out.

---

# Expected Placeholder Data

The complete 128-byte test payload is:

```text
00 01 02 03 04 05 06 07
08 09 0A 0B 0C 0D 0E 0F
10 11 12 13 14 15 16 17
18 19 1A 1B 1C 1D 1E 1F
20 21 22 23 24 25 26 27
28 29 2A 2B 2C 2D 2E 2F
30 31 32 33 34 35 36 37
38 39 3A 3B 3C 3D 3E 3F
40 41 42 43 44 45 46 47
48 49 4A 4B 4C 4D 4E 4F
50 51 52 53 54 55 56 57
58 59 5A 5B 5C 5D 5E 5F
60 61 62 63 64 65 66 67
68 69 6A 6B 6C 6D 6E 6F
70 71 72 73 74 75 76 77
78 79 7A 7B 7C 7D 7E 7F
```

The purpose of this pattern is to make it easy to verify that all 128 bytes are preserved through the complete transmission path.

---

# Build

After making the three required configuration changes:

```bash
cd ~/Desktop/premdai_code/STM32WL55_M0plus_standalone

make clean
make
```

If the project uses a specific existing build or flash target, continue using that target after applying the source and configuration changes.

---

# Transmission Chain

The final implementation is:

```text
Telemetry_GetReceivedData()
             │
             ▼
     128-byte payload
             │
             ▼
 Protocol_CreatePacket()
             │
             ▼
        AX.25 frame
             │
             ▼
       G3RUH scrambling
             │
             ▼
          GMSK
             │
             ▼
          SX1262
             │
             ▼
        RF transmission
```

The existing radio, G3RUH, and SX1262 transmission implementation is retained for this test.

---

# Files That Should Not Be Changed

For the placeholder-only test, do not modify:

```text
radio_app.c
radio_driver/
protocol/g3ruh/
ipc/Sring_buffer.c
ipc/Sring_buffer.h
ipc/ringbuffer_main.c
ipc/ipcc_m0plus_main.c
```

unless a separate build or integration issue specifically requires it.

---

# Future M4 Integration

When the real camera data path is ready, the placeholder implementation inside:

```c
Telemetry_GetReceivedData()
```

will be replaced with the M4 → shared SRAM/ring-buffer reader.

The public interface can remain unchanged:

```c
bool Telemetry_GetReceivedData(
    uint8_t *buffer,
    uint16_t bufferSize,
    uint16_t *dataLength);
```

The intended future architecture is:

```text
Camera
   │
   ▼
M4
   │
   ▼
Shared SRAM / Ring Buffer
   │
   ▼
M0+
   │
   ▼
Telemetry_GetReceivedData()
   │
   ▼
128-byte telemetry
   │
   ▼
AX.25
   │
   ▼
G3RUH
   │
   ▼
GMSK
   │
   ▼
SX1262
```

> **Important:** The current IPC ring-buffer payload definition is not being expanded or enabled as part of this placeholder test. The real M4 integration should be handled separately.

---

# Test Objective

The objective of this version is to verify that a **complete 128-byte binary Information/Data field** can pass through the existing:

```text
M0+ → AX.25 → G3RUH → GMSK → SX1262
```

transmission path before introducing the real M4 camera/ring-buffer source.
pragatibasnet1234@gmail.com
