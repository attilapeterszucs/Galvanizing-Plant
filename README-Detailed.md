# PRJ-26001 · Galvanizing System PLC Controller

![WebVisu Interface](Images/WebVisuOverview.png)

![Project](https://img.shields.io/badge/Project-PRJ--26001-0078D4?style=flat-square)
![PLC](https://img.shields.io/badge/PLC-WAGO%20CC100%20751--9402-009B77?style=flat-square)
![IDE](https://img.shields.io/badge/IDE-CODESYS%20V3.5-FF6C00?style=flat-square)
![Standard](https://img.shields.io/badge/Standard-IEC%2060204--1-gray?style=flat-square)
![Status](https://img.shields.io/badge/Status-Issued%20for%20Construction-brightgreen?style=flat-square)

---

| Field | Value |
|---|---|
| **Client** | Howest - Cyber3Lab |
| **Site** | Building J - Cyber3Lab, Howest Brugge, Belgium |
| **Project Number** | PRJ-26001 |
| **PLC Platform** | WAGO Compact Controller 100 (751-9402) |
| **Programming Environment** | CODESYS V3.5 |
| **Control Voltage** | 24 VDC |
| **Power Supply** | 400 VAC, 3-Phase, 50 Hz |
| **Enclosure Rating** | IP65 (Field) / IP54 (Panel) |
| **Standard** | IEC 60204-1 / AS/NZS 3000 |

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Production Line Layout](#2-production-line-layout)
3. [Hardware Architecture](#3-hardware-architecture)
4. [Electrical Documentation (ePlan)](#4-electrical-documentation-eplan)
5. [Software Architecture](#5-software-architecture)
6. [Global Variable List (GVL)](#6-global-variable-list-gvl)
7. [PLC_PRG - State Machine Reference](#7-plc_prg---state-machine-reference)
8. [Hoist Control Logic](#8-hoist-control-logic)
9. [Process Parameter Control](#9-process-parameter-control)
10. [Alarm System](#10-alarm-system)
11. [Cabling and Wiring Documentation](#11-cabling-and-wiring-documentation)
12. [I/O Channel Map](#12-io-channel-map)
13. [Safety and Emergency Stop](#13-safety-and-emergency-stop)
14. [Simulation Mode](#14-simulation-mode)
15. [Modbus TCP Server](#15-modbus-tcp-server)
16. [Visualization and HMI](#16-visualization-and-hmi)
17. [Standards and Compliance](#17-standards-and-compliance)
18. [Project File Structure](#18-project-file-structure)
19. [Development Notes](#19-development-notes)

---

## 1. Project Overview

This project implements a fully automated **hot-dip galvanizing production line** controller. A single overhead hoist transports steel loads sequentially through a series of chemical bath and wash stations before delivering them to a dryer and final unload position.

The controller runs on a **WAGO CC100 (751-9402)** PLC programmed in **Structured Text (ST)** under **CODESYS V3.5**. It is designed for DIN-rail mounting inside an IP54-rated control panel and interfaces with field devices at IP65.

**Key design goals:**

> **Deadlock-free sequencing** via reverse-priority scanning (station 7 to 1)
>
> **Resume-capable emergency stop** that freezes all motion without losing state
>
> **Per-station process parameter control** with configurable setpoints, dual-limit alarms, and persistent RETAIN storage
>
> **Simulation mode** retained for WebVisu visualization and offline commissioning
>
> **Modbus TCP server** for external register-level access to all setpoints and status
>
> **Unified documentation** — ePlan schematics, CODESYS project, and Excel cabling package all share project reference PRJ-26001

---

## 2. Production Line Layout

```
 [0]        [1]       [2]       [3]      [4]        [5]       [6]      [7]      [8]
Start  > Acid Bath > Washer 1 > Rinse 1 > Zink Bath > Washer 2 > Rinse 2 > Dryer > End
(Load)     (HCl)      (Wash)    (Rinse)     (Zinc)     (Wash)    (Rinse)   (Dry) (Unload)
```

| Position  | Index | Description                   | Soak Time |
|-----------|:-----:|-------------------------------|:---------:|
| Start     | 0     | Load/input station (operator) | n/a       |
| Acid Bath | 1     | HCl acid pickling bath        | 10 s      |
| Washer 1  | 2     | First wash station            | 10 s      |
| Rinse 1   | 3     | First rinse station           | 10 s      |
| Zink Bath | 4     | Primary zinc galvanizing bath | 10 s      |
| Washer 2  | 5     | Second wash station           | 10 s      |
| Rinse 2   | 6     | Second rinse station          | 10 s      |
| Dryer     | 7     | Hot-air drying station        | 10 s      |
| End       | 8     | Unload / completion position  | n/a       |

> All station soak times default to `T#10S` and are independently configurable per station via the HMI bath settings dialog or Modbus Holding Register writes. Limits are clamped between `cSoakTimeMin` (1 s) and `cSoakTimeMax` (3600 s).

---

## 3. Hardware Architecture

### PLC - WAGO Compact Controller 100

| Property | Value |
|---|---|
| Model | WAGO CC100 - 751-9402 |
| Mounting | DIN Rail, IP20 |
| Supply Voltage | 24 VDC |
| Programming | CODESYS V3.5 (IEC 61131-3) |
| Connector Type | picoMAX CAGE CLAMP push-in |
| Protocols | Modbus TCP, EtherNet/IP, PROFINET, EtherCAT, BACnet, MQTT |
| On-board DI | 8 x 24 VDC (X12, sinking, 2.8 mA, RC filter 5 us) |
| On-board DIO | 8 x configurable DI/DO (X5) |
| On-board AI/AIO | 2 x AI + 2 x AIO (X6) |
| On-board NI/PT | 2 x NI1000/PT1000 (X13) |

### Network - WAGO Ethernet Switch

| Component | Model | Location | Connections |
|---|---|---|---|
| Ethernet Switch | WAGO 852-1812 | Network Cabinet | CC100 (X1), External Uplink, Service Port |

### Horizontal Axis - Stepper Motor Drive

| Component | Model | Notes |
|---|---|---|
| Linear Guide | 20x40 Belt-Driven, Nema 17 | Moves hoist left/right across 9 positions |
| Stepper Driver | SL2690A | 6.5 A, 1/128 microstep, 20-90 VDC motor supply |
| Control Interface | STEP/DIR/ENA | 24 VDC logic from CC100 DIO4-6 |
| Motor Supply | Dedicated 48 VDC PSU | Separate from 24 VDC logic rail |

> The SL2690A is controlled via STEP pulse toggling on every PLC scan cycle. DIP switches must be set to full-step or 1/2-step mode to maintain adequate pulse rate from the cyclic PLC task.

### Vertical Axis - DC Motor Drive

| Component | Model | Notes |
|---|---|---|
| DC Gear Motor | GY-370 (Open Circuit) | 24 VDC, 30 RPM, worm gearbox, self-locking on power loss |
| Motor Driver | 2x Finder 48 Series relay | DPDT, 24 VDC coil, 22.2 mA, DIN rail mounted |
| Wiring | Relay reversing circuit | Relay A = lower (forward), Relay B = lift (reverse) |
| Dead-time | 200 ms (software) | Enforced in PLC_PRG between direction changes |

> The worm gearbox provides mechanical self-locking when power is removed. The two relays swap motor polarity for up/down direction. A 200 ms software dead-time prevents both relay coils from energising simultaneously.

**Cable Schedule (Network):**

| Cable No. | From | From Port | To | To Port | Type | Length |
|:---------:|------|:---------:|-----|:-----:|:------:|:------:|
| W101 | WAGO 852-1812 | X3 | WAGO CC100 751-9402 | X1 | UTP | 0.5 m |
| W102 | WAGO 852-1812 | X1 | EXT\_NETWORK\_UPLINK | - | UTP | 3 m |
| W102 | WAGO 852-1812 | X2 | WAGO Web Panel 400 | X1 | UTP | 2 m |
| W103 | WAGO 852-1812 | X8 | EXT\_SERVICE\_CON | - | UTP | 2 m |

---

## 4. Electrical Documentation (ePlan)

The ePlan project `26001_Galvanizing_System_Interconnections` is organized into the following functional groups, all referencing panel cabinet **PCW1**:

```
26001_Galvanizing_System_Interconnections
|
+-- PWR  (PCW1)  Power Distribution
|   +-- Page 1: Power Supply Input 230V AC
|   +-- Page 2: Power Supply Input 230V AC (redundant / secondary)
|   +-- Page 3: 24V DC System Power Distribution
|   +-- Page 4: 24V DC Switch System Power
|   +-- Page 5: 24V DC PLC System Power
|
+-- MAIN (DOC)  Project Documentation
|   +-- Page 1: Title Page
|
+-- PLC  (PCW1)  PLC I/O Schematics
|   +-- Page 1: Digital Inputs 24V DC
|   +-- Page 2: Digital Outputs 24V DC
|   +-- Page 3: PLC Communication
|
+-- NET  (PCW1)  Network Infrastructure
|   +-- Page 1: Network Topology and Ethernet Switch
|   +-- Page 2: Network Topology and Ethernet Switch (cont.)
|
+-- HMI  (PCW1)  Operator Interface
    +-- Page 1: Main Operator Touch Panel
    +-- Page 2: Main Operator Touch Panel System Power
```

> All ePlan drawing references use the prefix `PRJ-26001`. Any changes to electrical topology must be reflected in both the ePlan schematics and the cabling Excel workbook.

---

## 5. Software Architecture

### CODESYS Project Structure

```
26001_Galvanizing_System_V1  (CODESYS Project)
|
+-- Alarm Configuration
|   +-- Error, Info, Warning  (alarm classes)
|   +-- ProcessAlarms         (15 alarm entries - upper and lower limits)
|   +-- AlarmStorage
|
+-- External
|   +-- ImagePool             (HOWEST_Logo)
|
+-- Main_Program
|   +-- EStopSequence (PRG)   Emergency stop blink logic
|   +-- PLC_PRG (PRG)         Main hoist controller
|
+-- Visual
|   +-- Dialogs
|   |   +-- DLG_BathSettings  Unified per-station settings dialog
|   +-- Alerts                Full alarm table
|   +-- Bath_Control          Process parameter control page
|   +-- Home                  Landing dashboard
|   +-- Main_Visu             Animated production overview
|   +-- Overview              Station status table
|
+-- GVL                       Global Variable List
|
+-- Task Configuration
    +-- AlarmManagerTask      Alarm engine (cyclic, 100 ms)
    +-- MainTask              PLC_PRG cyclic scan
    +-- VISU_TASK             Visualization rendering
```

### Language Usage

| Program Unit | Language | Purpose |
|---|---|---|
| `PLC_PRG` | Structured Text | Hoist sequencing, timers, counters, I/O, process parameters |
| `EStopSequence` | Structured Text | Emergency light 1 Hz blink pattern |
| `GVL` | Variable declaration | Shared I/O mapping and process setpoints across all POUs |

---

## 6. Global Variable List (GVL)

Attribute: `{attribute 'qualified_only'}` — all variables must be accessed as `GVL.<variable>`.

### Sensor Inputs

| Variable | Type | Description |
|---|---|---|
| `xStartSensor` | `BOOL` | Material detected at start position (pos 0) |
| `xEndSensor` | `BOOL` | Material delivered to end position (pos 8) |
| `xMasterSwitch` | `BOOL` | Production line enable switch (ANDed with xStartSensor) |
| `xStationOccupied` | `ARRAY[1..7] OF BOOL` | Station occupancy flags (1=Acid Bath to 7=Dryer) |

### Feedback Inputs

| Variable | Type | Description |
|---|---|---|
| `iActualPosition` | `INT` | Current hoist horizontal position (0=Start, 8=End) |
| `xIsFullyLifted` | `BOOL` | Hoist at top limit (set by vertical move timer) |
| `xIsFullyLowered` | `BOOL` | Hoist at bottom limit (set by vertical move timer) |

### Command Outputs

| Variable | Type | Description |
|---|---|---|
| `xLowerHoist` | `BOOL` | TRUE = lower hoist, FALSE = lift hoist |
| `xGrabLoad` | `BOOL` | TRUE = gripper/clamp closed, FALSE = open |
| `iTargetPosition` | `INT` | Commanded horizontal target position |

### Horizontal Axis (Stepper - SL2690A)

| Variable | Type | Description |
|---|---|---|
| `xStepPulse` | `BOOL` | DO — PUL+ on SL2690A (CC100 DIO X5_4) |
| `xStepDir` | `BOOL` | DO — DIR+ on SL2690A, TRUE = right (CC100 DIO X5_5) |
| `xStepEnable` | `BOOL` | DO — ENA+ on SL2690A (CC100 DIO X5_6) |

### Vertical Axis (DC Motor - GY-370 via relays)

| Variable | Type | Description |
|---|---|---|
| `xRelayDown` | `BOOL` | DO — Relay A coil, energises to lower hoist (CC100 DIO X5_7) |
| `xRelayUp` | `BOOL` | DO — Relay B coil, energises to lift hoist (CC100 DIO X5_8) |

### Safety and Lights

| Variable | Type | Description |
|---|---|---|
| `xEmergencyStop` | `BOOL` | Global emergency stop — freezes all motion outputs |
| `xEmergencyLight` | `BOOL` | Blinks at 1 Hz while E-stop is active |
| `xInProcessLight` | `BOOL` | TRUE when any station is occupied or hoist carries load |

### Alarm Flags

| Variable | Type | Description |
|---|---|---|
| `xWashFlowAlarm` | `ARRAY[1..4] OF BOOL` | TRUE if wash flow exceeds upper limit |
| `xWashFlowAlarmLow` | `ARRAY[1..4] OF BOOL` | TRUE if wash flow drops below lower limit |
| `xAcidAlarm` | `BOOL` | TRUE if acid pH exceeds upper limit |
| `xAcidAlarmLow` | `BOOL` | TRUE if acid pH drops below lower limit |
| `xZincAlarm` | `BOOL` | TRUE if zinc temperature exceeds upper limit |
| `xZincAlarmLow` | `BOOL` | TRUE if zinc temperature drops below lower limit |
| `xDryerAlarm` | `BOOL` | TRUE if dryer temperature exceeds upper limit |
| `xDryerAlarmLow` | `BOOL` | TRUE if dryer temperature drops below lower limit |

### Bath Settings Dialog (shared INT — stations 2 through 7)

| Variable | Type | Description |
|---|---|---|
| `iSettingInput` | `INT` | Operator-typed setting value |
| `iSettingMin` | `INT` | Dynamic min clamp for current station |
| `iSettingMax` | `INT` | Dynamic max clamp for current station |
| `iSettingAlarmLim` | `INT` | Operator-editable upper alarm limit |
| `iSettingAlarmLimLow` | `INT` | Operator-editable lower alarm limit |
| `iSettingDefault` | `INT` | Operator-editable default value |
| `sSettingLabel` | `STRING` | Dynamic label shown in dialog (e.g. "Flow (L/min)") |

### Bath Settings Dialog (REAL — Acid Bath station 1 only)

| Variable | Type | Description |
|---|---|---|
| `rAcidSettingInput` | `REAL` | Operator-typed pH value |
| `rAcidSettingMin` | `REAL` | Dynamic min clamp |
| `rAcidSettingMax` | `REAL` | Dynamic max clamp |
| `rAcidSettingAlarmLim` | `REAL` | Upper alarm limit |
| `rAcidSettingAlarmLimLow` | `REAL` | Lower alarm limit |
| `rAcidSettingDefault` | `REAL` | Default pH value |

### Soak Time Configuration

| Variable | Type | Description |
|---|---|---|
| `iSelectedStation` | `INT` | Station index selected via HMI tile (1-7) |
| `sSelectedStation` | `STRING` | Human-readable station name shown in dialog title |
| `iSoakTimeInput` | `INT` | Operator-entered soak time in seconds |
| `xConfirmSoakTime` | `BOOL` | Confirm trigger from HMI; PLC clears after applying |
| `iSoakTimeSec` | `ARRAY[1..7] OF WORD` | Soak times in seconds, converted for Modbus export |

### Modbus WORD Mirrors (Input Registers)

| Variable | Type | Source | Description |
|---|---|---|---|
| `wActiveLoads` | `WORD` | `PLC_PRG.iActiveLoads` | Loads currently in system |
| `wTotalDone` | `WORD` | `PLC_PRG.iTotalDone` | Completed load counter |
| `wAlarmWord` | `WORD` | `PLC_PRG.wAlarmWord` | Packed alarm bits (see Section 10) |
| `wLowerHoist` | `WORD` | `GVL.xLowerHoist` | Current vertical command |
| `wGrabLoad` | `WORD` | `GVL.xGrabLoad` | Current gripper state |
| `wStepEnable` | `WORD` | `GVL.xStepEnable` | Stepper driver enabled status |
| `wRelayDown` | `WORD` | `GVL.xRelayDown` | Lower relay active status |
| `wRelayUp` | `WORD` | `GVL.xRelayUp` | Lift relay active status |
| `wSystemState` | `WORD` | `PLC_PRG.iSystemState` | State machine current state |

### VAR_GLOBAL RETAIN (persist across power cycles)

All operator-configured process setpoints and limits are declared `VAR_GLOBAL RETAIN`. They survive power cycles and warm restarts. A cold download with reset is required to restore factory defaults.

| Group | Variables |
|---|---|
| Soak times | `ptStationTime[1..7]` |
| Wash flow | `iWashFlowSetpoint[1..4]`, `cWashFlowDefault`, `cWashFlowAlarmLim`, `cWashFlowAlarmLimLow`, `cWashFlowMax`, `cWashFlowMin` |
| Acid Bath (REAL) | `iAcidConcentration`, `cAcidDefault`, `cAcidAlarmLim`, `cAcidAlarmLimLow`, `cAcidMax`, `cAcidMin` |
| Zinc Bath | `iZincTemperature`, `cZincDefault`, `cZincAlarmLim`, `cZincAlarmLimLow`, `cZincMax`, `cZincMin` |
| Dryer | `iDryerTemperature`, `cDryerDefault`, `cDryerAlarmLim`, `cDryerAlarmLimLow`, `cDryerMax`, `cDryerMin` |
| Hardware outputs | `xRelayDown`, `xRelayUp`, `xStepPulse`, `xStepDir`, `xStepEnable` |

---

## 7. PLC_PRG - State Machine Reference

The main program executes the following sections on every PLC scan cycle:

| Section | Description |
|:-------:|---|
| 1 | Run station timers (TON, configurable per station via `ptStationTime`) |
| 2 | Set `xHoistBusy` flag (TRUE when state is not 0) |
| 3 | Emergency stop check + main CASE state machine |
| 4 | Real I/O: vertical axis relay reversal with dead-time |
| 4b | Real I/O: horizontal axis stepper pulse generation |
| 4c | Simulation: position update for WebVisu visualization |
| 5 | Count active loads in system (occupied stations + gripper) |
| 5b | Drive in-process lamp from active load count |
| 6 | Count completed loads (rising-edge on `xEndSensor`) |
| 7 | Reset done counter (from HMI button) |
| 8 | Get system time via `SysTimeRtcGet` |
| 9 | Process parameter clamp and dual-limit alarm generation |
| 9e | Modbus WORD sync (all mirror variables updated) |
| 10 | Bath settings configuration from HMI dialog confirm |
| 11 | Call `EStopSequence()` |

### State Machine States

| State | Name | Action |
|:-----:|---|---|
| `0` | IDLE / Scan | Reverse-priority scan (station 7 to 1 to start) for the next job |
| `10` | Move to Pickup | Wait until `iActualPosition = iTargetPosition` |
| `20` | Lower at Pickup | Assert `xLowerHoist`; when `xIsFullyLowered`, close gripper |
| `25` | Grab Confirmation | One-scan hold with hoist lowered — ensures gripper engagement |
| `30` | Lift with Load | De-assert `xLowerHoist`; clears station occupancy when fully lifted |
| `40` | Move to Drop-off | Wait until hoist reaches destination position |
| `45` | Lower and Release | Assert `xLowerHoist`; when lowered, open gripper, set occupancy |
| `50` | Lift Empty / Return | De-assert `xLowerHoist`; returns to IDLE when fully lifted |

### Priority Scheme (State 0 - IDLE)

| Priority | Condition | Action |
|:--------:|---|---|
| 1 | Station 7 done | Unload dryer — move to pos 8 (End) |
| 2 | Station 6 done, station 7 free | Advance station 6 forward |
| 3 | Station 5 done, station 6 free | Advance station 5 forward |
| 4 | Station 4 done, station 5 free | Advance station 4 forward |
| 5 | Station 3 done, station 4 free | Advance station 3 forward |
| 6 | Station 2 done, station 3 free | Advance station 2 forward |
| 7 | Station 1 done, station 2 free | Advance station 1 forward |
| 8 | `xMasterSwitch` AND `xStartSensor` AND station 1 free | Pick new batch from Start |

> This reverse-priority order is intentional. Later stations always get preference, ensuring the line drains before new material is loaded.

---

## 8. Hoist Control Logic

### Full Cycle for One Load

```
IDLE
 |
 +-> [State 10] Move to pickup position (stepper pulses until timer expires)
 |
 +-> [State 20] Lower hoist (Relay A energised for T#2S)
 |
 +-> [State 25] Grip confirmation (one-scan hold)
 |
 +-> [State 30] Lift with load (Relay B energised for T#2S)
 |
 +-> [State 40] Move to drop-off position (stepper pulses until timer expires)
 |
 +-> [State 45] Lower and release (Relay A energised for T#2S)
 |
 +-> [State 50] Lift empty (Relay B energised for T#2S)
 |
IDLE
```

### Motion Timing

| Axis | Timer | Default | Notes |
|---|---|:---:|---|
| Horizontal (one station hop) | `fbHorizontalMove` | `T#3S` | Tune against real belt travel time |
| Vertical (full stroke) | `fbVerticalMove` | `T#2S` | Tune against real motor stroke time |
| Relay direction dead-time | `fbVerticalDeadTime` | `T#200MS` | Do not reduce below 100 ms |

### Occupancy Rules

| Event | Occupancy Change |
|---|---|
| Load lifted from station N | `xStationOccupied[N] := FALSE` |
| Load placed at station N | `xStationOccupied[N] := TRUE` |
| Load placed at position 8 | `xEndSensor := TRUE` (triggers done counter) |
| Load picked from Start (pos 0) | No occupancy array entry for pos 0 |

---

## 9. Process Parameter Control

Section 9 of `PLC_PRG` runs every scan and enforces hard clamps and dual-limit alarm flags on all operator-adjustable process setpoints. Operators configure values via the `DLG_BathSettings` dialog, which opens when any station tile is clicked on the Bath_Control page. The dialog is unified across all stations and dynamically populates its fields from the selected station's GVL variables, writing back on Confirm.

### Wash Flow (Washers and Rinse Stations)

| Index | Station | Default | Alarm High | Alarm Low | Max | Min | Unit |
|:-----:|---|:---:|:---:|:---:|:---:|:---:|:---:|
| `[1]` | Washer 1 (pos 2) | 30 | 40 | 22 | 40 | 20 | L/min |
| `[2]` | Rinse 1 (pos 3) | 30 | 40 | 22 | 40 | 20 | L/min |
| `[3]` | Washer 2 (pos 5) | 30 | 40 | 22 | 40 | 20 | L/min |
| `[4]` | Rinse 2 (pos 6) | 30 | 40 | 22 | 40 | 20 | L/min |

### Acid Bath — pH (REAL type, supports decimal values)

| Parameter | Value |
|---|:---:|
| Default | 4.5 |
| Alarm High | 4.9 |
| Alarm Low | 4.1 |
| Max | 4.8 |
| Min | 4.2 |

> Acid bath variables use `REAL` type throughout to support decimal pH values (e.g. 4.5). All other station parameters use `INT`. The confirm path in Section 10 handles acid bath exclusively through the `rAcidSetting*` REAL variables — the shared INT clamp is skipped for station 1.

### Zinc Bath — Temperature (°C)

| Parameter | Value |
|---|:---:|
| Default | 60 |
| Alarm High | 65 |
| Alarm Low | 52 |
| Max | 65 |
| Min | 50 |

### Dryer — Air Temperature (°C)

| Parameter | Value |
|---|:---:|
| Default | 70 |
| Alarm High | 80 |
| Alarm Low | 55 |
| Max | 80 |
| Min | 50 |

---

## 10. Alarm System

Alarms are managed by the CODESYS `AlarmManager` running in `AlarmManagerTask` (cyclic, 100 ms). All alarms are defined in `ProcessAlarms` under Alarm Configuration.

### Alarm Classes

| Class | Color | Used For |
|---|---|---|
| `Error` | Red | Emergency stop |
| `Warning` | Yellow | All process parameter limit violations |

### Alarm Entries (ProcessAlarms — 15 total)

| ID | Trigger Variable | Class | Message |
|---|---|---|---|
| `ALM_EStop` | `GVL.xEmergencyStop` | Error | Emergency stop is active |
| `ALM_Acid` | `GVL.xAcidAlarm` | Warning | Acid Bath: HCl concentration exceeded safe limit |
| `ALM_AcidLow` | `GVL.xAcidAlarmLow` | Warning | Acid Bath: pH dropped below minimum safe limit |
| `ALM_Zinc` | `GVL.xZincAlarm` | Warning | Zinc Bath: Temperature exceeded safe limit |
| `ALM_ZincLow` | `GVL.xZincAlarmLow` | Warning | Zinc Bath: Temperature dropped below minimum safe limit |
| `ALM_Dryer` | `GVL.xDryerAlarm` | Warning | Dryer: Air temperature exceeded safe limit |
| `ALM_DryerLow` | `GVL.xDryerAlarmLow` | Warning | Dryer: Temperature dropped below minimum safe limit |
| `ALM_WashFlow1` | `GVL.xWashFlowAlarm[1]` | Warning | Washer No.1: Flow rate exceeded safe limit |
| `ALM_WashFlow1Low` | `GVL.xWashFlowAlarmLow[1]` | Warning | Washer No.1: Flow rate dropped below minimum limit |
| `ALM_WashFlow2` | `GVL.xWashFlowAlarm[2]` | Warning | Washer No.2: Flow rate exceeded safe limit |
| `ALM_WashFlow2Low` | `GVL.xWashFlowAlarmLow[2]` | Warning | Washer No.2: Flow rate dropped below minimum limit |
| `ALM_RinseFlow1` | `GVL.xWashFlowAlarm[3]` | Warning | Rinse No.1: Flow rate exceeded safe limit |
| `ALM_RinseFlow1Low` | `GVL.xWashFlowAlarmLow[3]` | Warning | Rinse No.1: Flow rate dropped below minimum limit |
| `ALM_RinseFlow2` | `GVL.xWashFlowAlarm[4]` | Warning | Rinse No.2: Flow rate exceeded safe limit |
| `ALM_RinseFlow2Low` | `GVL.xWashFlowAlarmLow[4]` | Warning | Rinse No.2: Flow rate dropped below minimum limit |

### Alarm Word Bit Map (wAlarmWord — Modbus Input Register 2)

| Bit | Source Variable | Condition |
|:---:|---|---|
| 0 | `GVL.xWashFlowAlarm[1]` | Wash 1 flow too high |
| 1 | `GVL.xWashFlowAlarm[2]` | Wash 2 flow too high |
| 2 | `GVL.xWashFlowAlarm[3]` | Wash 3 flow too high |
| 3 | `GVL.xWashFlowAlarm[4]` | Wash 4 flow too high |
| 4 | `GVL.xAcidAlarm` | Acid pH too high |
| 5 | `GVL.xZincAlarm` | Zinc temp too high |
| 6 | `GVL.xDryerAlarm` | Dryer temp too high |
| 7 | `GVL.xEmergencyStop` | E-stop active |
| 8 | `GVL.xWashFlowAlarmLow[1]` | Wash 1 flow too low |
| 9 | `GVL.xWashFlowAlarmLow[2]` | Wash 2 flow too low |
| 10 | `GVL.xWashFlowAlarmLow[3]` | Wash 3 flow too low |
| 11 | `GVL.xWashFlowAlarmLow[4]` | Wash 4 flow too low |
| 12 | `GVL.xAcidAlarmLow` | Acid pH too low |
| 13 | `GVL.xZincAlarmLow` | Zinc temp too low |
| 14 | `GVL.xDryerAlarmLow` | Dryer temp too low |
| 15 | — | Reserved |

### HMI Alarm Display

An **Alarm Banner** sits at the top of every navigation page (Home, Main_Visu, Bath_Control, Overview). It scrolls through active alarms automatically when multiple are active simultaneously. The **Alerts** page hosts a full **Alarm Table** with Timestamp, State, Class, and Message columns. `ALM_EStop` is configured as a latched alarm requiring manual operator acknowledgement.

---

## 11. Cabling and Wiring Documentation

The Excel workbook **`PRJ-26001_Galvanizing-System_PLC-Cabling-Wiring.xlsx`** contains seven sheets:

| Sheet | Contents |
|---|---|
| `Config` | Central variable definitions — project name, PLC model, voltages, I/O map |
| `Cover` | Project cover page, document index, and general installation notes |
| `Cable Schedule` | Full cable listing: number, source, destination, type, cores, size, voltage, shield, length, conduit |
| `IO List` | PLC I/O assignments: tag, description, cable/wire number, connector, CODESYS address, signal type |
| `Terminal Block Layout` | DIN rail terminal assignments: number, type, wire color, cross-section, ferrule, jumper notes |
| `Wiring Legend` | IEC 60446 wire color codes, cable type specifications, abbreviations, standards references |
| `Cable Drum Register` | Drum tracking: ID, cable type, full length, issued length, remaining, location, supplier |

### Wire Color Reference (IEC 60446)

| Function | Color | Voltage | Cross-Section |
|---|---|:---:|:---:|
| 24V DC Positive | Red | 24V DC | 20 AWG / 0.5 mm2 |
| 0V DC Negative | Black | 0V DC | 20 AWG / 0.5 mm2 |
| Protective Earth (PE) | Green/Yellow | Earth | 20 AWG / 0.5 mm2 |
| AC Line (L) | Brown | 230V AC | 3G 1 mm2 |
| AC Neutral (N) | Blue | 0V AC | 3G 1 mm2 |

### General Wiring Rules

- All cabling per **IEC 60204-1** and local wiring regulations
- Signal cables (4-20 mA) must be segregated from power cables — **minimum 200 mm separation**
- All cable screens/shields grounded at **one end only** (control panel end) unless noted otherwise
- Minimum cable bending radius: **6x OD** (multi-core), **8x OD** (armoured)
- All field terminations use **bootlace ferrules** (Weidmuller crimping tool or equivalent)
- Motor cables must be shielded (VFD-duty) where inverter drives are installed

---

## 12. I/O Channel Map

### WAGO CC100 On-board I/O Connector Map

| Connector | Channels | Type | CODESYS Prefix | Notes |
|:---------:|---|---|:---:|---|
| X11 | DI.1 to DI.8 | Digital Input | `%IX0.0 to IX0.7` | 24 VDC sinking, 2.8 mA, RC filter 5 us |
| X5 | DIO.1 to DIO.8 | Configurable DIO | `%IX6.x / QX` | Configurable as DI or DO |
| X6 | AI.1-AI.2 + AIO.1-AIO.2 | Analog | `%IW / %QW` | 4-20 mA or 0-10 V |
| X13 | NI/PT.1 to NI/PT.2 | Temperature | `%IW` | NI1000 / PT1000 |

### DIO Assignments (X5 - Configured as DO)

| Terminal | CODESYS Variable | Direction | Connected To |
|:---:|---|:---:|---|
| DIO4 (X5_4) | `GVL.xStepPulse` | DO | SL2690A PUL+ |
| DIO5 (X5_5) | `GVL.xStepDir` | DO | SL2690A DIR+ |
| DIO6 (X5_6) | `GVL.xStepEnable` | DO | SL2690A ENA+ |
| DIO7 (X5_7) | `GVL.xRelayDown` | DO | Relay A coil + (lower) |
| DIO8 (X5_8) | `GVL.xRelayUp` | DO | Relay B coil + (lift) |

> All SL2690A signal commons (PUL-, DIR-, ENA-) connect to 24V GND. Relay coil negatives connect to 24V GND.

---

## 13. Safety and Emergency Stop

### Emergency Stop Behavior

When `GVL.xEmergencyStop = TRUE`:

| Output | Forced Value | Effect |
|---|:---:|---|
| `GVL.xLowerHoist` | `FALSE` | Hoist stops descending immediately |
| `GVL.xGrabLoad` | `FALSE` | Gripper deactivated |
| `GVL.iTargetPosition` | frozen | No horizontal motion commanded |
| `GVL.xRelayDown` | `FALSE` | Lower relay de-energised |
| `GVL.xRelayUp` | `FALSE` | Lift relay de-energised |
| `GVL.xStepPulse` | `FALSE` | Stepper pulse stops |
| `iSystemState` | unchanged | State machine resumes on E-stop release |
| `GVL.xEmergencyLight` | blinking | 1 Hz blink via EStopSequence |

> The GY-370 worm gearbox provides mechanical self-locking when relay power is removed, holding the load in place during an E-stop. After an E-stop, the operator must manually verify the hoist position and load status before releasing. `ALM_EStop` is a latched alarm — it remains active in the alarm table until the operator explicitly acknowledges it even after the E-stop is cleared.

### EStopSequence Program

`EStopSequence` is a standalone PRG called at the end of every scan. It uses a single self-resetting `TON` timer to produce a 1 Hz, 50% duty-cycle blink on `GVL.xEmergencyLight` (ON for 500 ms, OFF for 500 ms). The lamp turns off immediately when the E-stop is cleared.

---

## 14. Simulation Mode

Section 4c of `PLC_PRG` retains simulation logic that runs in parallel with the real I/O sections (4 and 4b). This allows the WebVisu visualization to reflect hoist movement and station occupancy without requiring physical hardware.

### Simulation Behavior

| Simulated Signal | Behavior |
|---|---|
| Horizontal position | `iActualPosition` increments/decrements by 1 per scan toward `iTargetPosition` |
| Vertical (lowered) | `xIsFullyLowered := xLowerHoist` (instant) |
| Vertical (lifted) | `xIsFullyLifted := NOT xLowerHoist` (instant) |

> The simulation section does not need to be removed for live hardware deployment. Real feedback is provided by the vertical and horizontal move timers in sections 4 and 4b. The simulation section runs harmlessly alongside and only affects WebVisu display variables.

---

## 15. Modbus TCP Server

A Modbus TCP Server device is configured in the CODESYS project under `ModbusTCP_Server_Device`. It exposes 9 Holding Registers (writable) and 16 Input Registers (read-only).

### Holding Registers (Writable — external client to PLC)

| Register | Address | Variable | Description |
|:---:|:---:|---|---|
| 0 | %IW8 | `GVL.iWashFlowSetpoint[1]` | Wash 1 flow setpoint (L/min) |
| 1 | %IW9 | `GVL.iWashFlowSetpoint[2]` | Wash 2 flow setpoint (L/min) |
| 2 | %IW10 | `GVL.iWashFlowSetpoint[3]` | Wash 3 flow setpoint (L/min) |
| 3 | %IW11 | `GVL.iWashFlowSetpoint[4]` | Wash 4 flow setpoint (L/min) |
| 4 | %IW12 | `GVL.iZincTemperature` | Zinc bath temperature (°C) |
| 5 | %IW13 | `GVL.iDryerTemperature` | Dryer air temperature (°C) |
| 6 | %IW14 | `GVL.iSelectedStation` | Station index for settings dialog (1-7) |
| 7 | %IW15 | `GVL.iSoakTimeInput` | Soak time to apply in seconds |
| 8 | %IW16 | `GVL.xConfirmSoakTime` | Write 1 to confirm soak time update |

> `GVL.iAcidConcentration` is `REAL` type and cannot be mapped directly to a WORD Holding Register. Remote acid bath pH adjustment via Modbus requires a UNION-based register pair or integer scaling and is flagged as a known limitation.

### Input Registers (Read-only — PLC to external client)

| Register | Address | Variable | Description |
|:---:|:---:|---|---|
| 0 | %QW1 | `GVL.wActiveLoads` | Number of loads currently in system |
| 1 | %QW2 | `GVL.wTotalDone` | Completed load counter |
| 2 | %QW3 | `GVL.wAlarmWord` | Packed alarm bits (see Section 10) |
| 3 | %QW4 | `GVL.wRelayDown` | Lower relay active status |
| 4 | %QW5 | `GVL.wRelayUp` | Lift relay active status |
| 5 | %QW6 | `GVL.wStepEnable` | Stepper driver enabled status |
| 6 | %QW7 | `GVL.iSoakTimeSec[1]` | Station 1 soak time (seconds) |
| 7 | %QW8 | `GVL.iSoakTimeSec[2]` | Station 2 soak time (seconds) |
| 8 | %QW9 | `GVL.iSoakTimeSec[3]` | Station 3 soak time (seconds) |
| 9 | %QW10 | `GVL.iSoakTimeSec[4]` | Station 4 soak time (seconds) |
| 10 | %QW11 | `GVL.iSoakTimeSec[5]` | Station 5 soak time (seconds) |
| 11 | %QW12 | `GVL.iSoakTimeSec[6]` | Station 6 soak time (seconds) |
| 12 | %QW13 | `GVL.iSoakTimeSec[7]` | Station 7 soak time (seconds) |
| 13 | %QW14 | `GVL.wLowerHoist` | Current vertical command (1=lower) |
| 14 | %QW15 | `GVL.wGrabLoad` | Current gripper state (1=closed) |
| 15 | %QW16 | `GVL.wSystemState` | State machine current state |

---

## 16. Visualization and HMI

The project includes five CODESYS visualization pages accessible via both a physical **TargetVisu** WAGO panel and a browser-based **WebVisu** interface over the WAGO 852-1812 switch network.

| Page | Purpose |
|---|---|
| `Home` | Landing dashboard — system status, live clock, master switch, E-stop indicator |
| `Main_Visu` | Animated production overview — hoist position, station occupancy, load counts |
| `Overview` | Station status table — occupied flags, active loads, completed batches, reset button |
| `Bath_Control` | Per-station process parameter control — settings dialog, live values |
| `Alerts` | Full alarm table with Timestamp, State, Class, Message columns |

### Core Variable Bindings

| UI Element | Variable | Function |
|---|---|---|
| Station indicators | `GVL.xStationOccupied[1..7]` | Show occupied/free status |
| Hoist position | `GVL.iActualPosition` | Numeric display / graphic |
| Active loads counter | `PLC_PRG.iActiveLoads` | Count of loads currently in system |
| Completed loads | `PLC_PRG.iTotalDone` | Running total of finished batches |
| Reset counter button | `PLC_PRG.xResetDoneCounter` | Resets `iTotalDone` to 0 |
| Emergency stop | `GVL.xEmergencyStop` | E-stop indicator / button |
| Master switch | `GVL.xMasterSwitch` | Production enable toggle |
| In-process lamp | `GVL.xInProcessLight` | Green lamp, on when loads in system |
| Emergency light | `GVL.xEmergencyLight` | Blinks at 1 Hz during E-stop |
| System clock | `PLC_PRG.sFormattedTime` | Live HH:MM:SS display |
| Alarm Banner | `ProcessAlarms` | Scrolls active alarms on all pages |
| Alarm Table | `ProcessAlarms` | Full alarm list on Alerts page |

### DLG_BathSettings Dialog

The unified bath settings dialog opens when any station tile is clicked on the Bath_Control page. It pre-populates all fields from the selected station's GVL variables and writes back on Confirm. Fields shown: Time (seconds), current process value, Min, Max, Default, Alarm Limit (High), Alarm Limit (Low). Acid bath fields use `%.1f` format to display decimal pH values; all other stations use integer display.

---

## 17. Standards and Compliance

| Standard | Application |
|---|---|
| IEC 61131-3 | PLC programming languages (ST, LD, FBD, SFC, IL) |
| IEC 60204-1 | Safety of machinery — electrical equipment |
| IEC 60446 | Wire identification by color codes |
| IEC 60529 | Enclosure IP rating classification |
| AS/NZS 3000 | Wiring rules (supplementary local reference) |

---

## 18. Project File Structure

```
PRJ-26001_Galvanizing_System/
|
+-- CODESYS/
|   +-- 26001_Galvanizing_System_V1.project    Main CODESYS project file
|   +-- GVL.gvl                                Global Variable List
|   +-- PLC_PRG.st                             Main ST program
|   +-- EStopSequence.st                       Emergency stop blink program
|
+-- ePlan/
|   +-- 26001_Galvanizing_System_Interconnections/
|       +-- PWR/     Power supply schematics (pages 1-5)
|       +-- MAIN/    Title page
|       +-- PLC/     I/O schematics (pages 1-3)
|       +-- NET/     Network topology (pages 1-2)
|       +-- HMI/     Touch panel wiring (pages 1-2)
|
+-- Documentation/
|   +-- PRJ-26001_Galvanizing-System_PLC-Cabling-Wiring.xlsx
|   +-- README.md
|
+-- Datasheets/
    +-- WAGO_CC100_751-9402_Datasheet.pdf
    +-- WAGO_852-1812_Switch_Datasheet.pdf
    +-- SL2690A_Stepper_Driver_Datasheet.pdf
    +-- Finder_48_Series_Relay_Datasheet.pdf
    +-- GY370_DC_Gearmotor_Datasheet.pdf
```

---

## 19. Development Notes

### Known Limitations

- **Horizontal hop time** `T#3S` in `fbHorizontalMove` must be measured and tuned on real hardware. One value covers all station hops, assuming uniform belt speed and equal spacing between positions.
- **Vertical stroke time** `T#2S` in `fbVerticalMove` must be measured and tuned on real hardware for both lower and lift strokes.
- **No physical position sensors.** The system relies entirely on timed motion for both axes. If belt slip or motor stall occurs, `iActualPosition` will drift from the real physical position. For production use, proximity sensors at each station plus top/bottom limit switches are recommended.
- **Single-hoist only.** The state machine is designed for one hoist. Multi-hoist operation would require a significant redesign with collision avoidance logic.
- **`GVL.xEndSensor`** is written by the PLC itself (set in State 45 when position = 8). On real hardware this should be an actual field sensor and the software write should be removed.
- **Acid bath Modbus write limitation.** `iAcidConcentration` is `REAL` type and cannot be mapped directly to a Modbus WORD Holding Register. Remote pH adjustment via Modbus requires a UNION-based register pair or integer scaling.
- **Alarm acknowledgement.** Warning-class alarms auto-clear when their trigger variable goes FALSE. `ALM_EStop` is latched and requires manual acknowledgement. For a production deployment, consider latching all process alarms per IEC 60204-1 requirements.
- **`Main_LD()`** is called but may be empty. Populate with hardwired interlock logic before commissioning.
- **SL2690A motor supply** requires a dedicated 48 VDC PSU — the existing 24 VDC logic rail is insufficient for the stepper motor power stage.

### Commissioning Checklist

- [ ] Tune `T#3S` horizontal hop time against real belt travel measurement
- [ ] Tune `T#2S` vertical stroke time against real motor travel measurement
- [ ] Verify DIO4-8 output wiring against terminal block layout sheet
- [ ] Confirm 48 VDC PSU installed and wired to SL2690A motor power terminals
- [ ] Confirm SL2690A DIP switches set to full-step or 1/2-step mode
- [ ] Confirm relay A and relay B interlock wiring (hardware and software)
- [ ] Verify all GVL variables mapped to physical I/O in CODESYS device tree
- [ ] Verify terminal block wiring against `Terminal Block Layout` sheet
- [ ] Update `Cable Schedule` sheet with all final cable lengths and routes
- [ ] Confirm E-stop circuit wiring per ePlan PWR schematics
- [ ] Test `Main_LD` interlock logic independently before enabling `xMasterSwitch`
- [ ] Set production soak times per process engineer specification
- [ ] Validate HMI connectivity (WebVisu) over WAGO switch
- [ ] Validate Modbus TCP register map against external SCADA or test client
- [ ] Perform dry-run cycle with no loads before chemical commissioning

### Contact / Authorship

| Role | Detail |
|---|---|
| Client | Howest - Cyber3Lab |
| Site | Building J, Howest Brugge |
| Project Reference | PRJ-26001 |
| PLC Supplier | WAGO (wago.com) |
| Software | CODESYS V3.5 |

---

*Last updated: 2026 | Rev C - Process parameter control, dual-limit alarm system, acid bath REAL type, Modbus WORD mirrors, and unified bath settings dialog by Attila Peter Szucs*