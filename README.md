# PRJ-26001 · Galvanizing System PLC Controller

![WebVisu Interface](images/WebVisu.png)

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
9. [Cabling and Wiring Documentation](#9-cabling-and-wiring-documentation)
10. [I/O Channel Map](#10-io-channel-map)
11. [Safety and Emergency Stop](#11-safety-and-emergency-stop)
12. [Simulation Mode](#12-simulation-mode)
13. [Visualization and HMI](#13-visualization-and-hmi)
14. [Standards and Compliance](#14-standards-and-compliance)
15. [Project File Structure](#15-project-file-structure)
16. [Development Notes](#16-development-notes)

---

## 1. Project Overview

This project implements a fully automated **hot-dip galvanizing production line** controller. A single overhead hoist transports steel loads sequentially through a series of chemical bath and wash stations before delivering them to a dryer and final unload position.

The controller is built on a **WAGO CC100 (751-9402)** PLC programmed entirely in **Structured Text (ST)** under **CODESYS V3.5**, with an auxiliary Ladder Diagram (LD) sub-program (`Main_LD`) for any hardwired interlock logic. The system is designed for DIN-rail mounting inside an IP54-rated control panel and interfaces with field devices at IP65.

**Key design goals:**

> **Deadlock-free sequencing** via reverse-priority scanning (station 7 to 1)
>
> **Resume-capable emergency stop** that freezes all motion without losing state
>
> **Simulation mode** for offline commissioning and lab testing without real I/O
>
> **Unified documentation** - ePlan schematics, CODESYS project, and Excel cabling package all share project reference PRJ-26001

---

## 2. Production Line Layout

```
 [0]      [1]      [2]      [3]      [4]      [5]      [6]      [7]      [8]
Start  > Bath1  > Wash1  > Wash2  > Bath2  > Wash3  > Wash4  > Dryer  > End
(Load)  (Zinc)  (Rinse) (Rinse)  (Flux)  (Rinse) (Rinse)   (Dry)  (Unload)
```

| Position | Index | Description                     | Soak Time |
|----------|:-----:|---------------------------------|:---------:|
| Start    | 0     | Load/input station (operator)   | n/a       |
| Bath 1   | 1     | Primary zinc galvanizing bath   | 10 s      |
| Wash 1   | 2     | First rinse station             | 10 s      |
| Wash 2   | 3     | Second rinse station            | 10 s      |
| Bath 2   | 4     | Flux / secondary treatment bath | 10 s      |
| Wash 3   | 5     | Third rinse station             | 10 s      |
| Wash 4   | 6     | Fourth rinse station            | 10 s      |
| Dryer    | 7     | Hot-air drying station          | 10 s      |
| End      | 8     | Unload / completion position    | n/a       |

> **Note:** All station soak/wash/dry times are currently set to `T#10S`. These are defined as `PT` parameters on `timerStation[1..7]` TON timers in `PLC_PRG` and can be adjusted independently per station for production tuning.

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

**Cable Schedule (Network):**

| Cable No. | From | From Port | To | Type | Length |
|:---------:|------|:---------:|-----|:----:|:------:|
| W101 | WAGO 852-1812 | X3 | WAGO CC100 751-9402 | UTP | 0.5 m |
| W102 | WAGO 852-1812 | X1 | EXT\_NETWORK\_UPLINK | UTP | 3 m |
| W103 | WAGO 852-1812 | X8 | EXT\_SERVICE\_CON | UTP | 2 m |

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
26001_Galvanizing_System  (CODESYS Project)
|
+-- GVL                  Global Variable List - shared I/O and flags
+-- PLC_PRG              Main program - Structured Text
|   +-- Station timers   TON array, 7 elements
|   +-- State machine    10-state hoist sequencer
|   +-- Simulation logic Removable for live I/O
|   +-- Load counters    Active + completed
|   +-- Main_LD()        Call to Ladder Diagram sub-program
+-- Main_LD              Ladder Diagram - hardwired interlocks
```

### Language Usage

| Program Unit | Language | Purpose |
|---|---|---|
| `PLC_PRG` | Structured Text | Hoist sequencing, timers, counters |
| `Main_LD` | Ladder Diagram | Hardwired safety interlocks (if retained) |
| `GVL` | Variable declaration | Shared I/O mapping across all POUs |

---

## 6. Global Variable List (GVL)

Attribute: `{attribute 'qualified_only'}` - all variables must be accessed as `GVL.<variable>`.

### Sensor Inputs

| Variable | Type | Description |
|---|---|---|
| `xStartSensor` | `BOOL` | Material detected at start position (pos 0) |
| `xEndSensor` | `BOOL` | Material delivered to end position (pos 8) |
| `xMasterSwitch` | `BOOL` | Production line enable switch (ANDed with xStartSensor) |
| `xStationOccupied` | `ARRAY[1..7] OF BOOL` | Station occupancy flags (1=Bath1 to 7=Dryer) |

### Feedback Inputs

| Variable | Type | Description |
|---|---|---|
| `iActualPosition` | `INT` | Current hoist horizontal position (0=Start, 8=End) |
| `xIsFullyLifted` | `BOOL` | Hoist at top limit (vertical feedback) |
| `xIsFullyLowered` | `BOOL` | Hoist at bottom limit / bath immersion sensor |

### Command Outputs

| Variable | Type | Description |
|---|---|---|
| `xLowerHoist` | `BOOL` | TRUE = lower hoist, FALSE = lift hoist |
| `xGrabLoad` | `BOOL` | TRUE = gripper/clamp closed, FALSE = open |
| `iTargetPosition` | `INT` | Commanded horizontal target position |

### Safety

| Variable | Type | Description |
|---|---|---|
| `xEmergencyStop` | `BOOL` | Global emergency stop - freezes all motion outputs |

---

## 7. PLC_PRG - State Machine Reference

The main program executes the following sections on every PLC scan cycle:

| Section | Description |
|:-------:|---|
| 1 | Run station timers (TON, 10 s per occupied station) |
| 2 | Set `xHoistBusy` flag (TRUE when state is not 0) |
| 3 | Emergency stop check + main CASE state machine |
| 4 | Simulation logic (override for offline testing) |
| 5 | Count active loads in system (occupied stations + gripper) |
| 6 | Count completed loads (rising-edge on `xEndSensor`) |
| 7 | Reset done counter (from visualization/HMI button) |
| 8 | Call `Main_LD()` sub-program |

### State Machine States

| State | Name | Action |
|:-----:|---|---|
| `0` | IDLE / Scan | Reverse-priority scan (station 7 to 1 to start) for the next job |
| `10` | Move to Pickup | Wait until `iActualPosition = iTargetPosition` |
| `20` | Lower at Pickup | Assert `xLowerHoist`; when `xIsFullyLowered`, close gripper |
| `25` | Grab Confirmation | One-scan hold with hoist lowered - ensures gripper engagement |
| `30` | Lift with Load | De-assert `xLowerHoist`; clears station occupancy when fully lifted |
| `40` | Move to Drop-off | Wait until hoist reaches destination position |
| `45` | Lower and Release | Assert `xLowerHoist`; when lowered, open gripper, set occupancy |
| `50` | Lift Empty / Return | De-assert `xLowerHoist`; returns to IDLE when fully lifted |

### Priority Scheme (State 0 - IDLE)

| Priority | Condition | Action |
|:--------:|---|---|
| 1 | Station 7 done | Unload dryer - move to pos 8 (End) |
| 2 | Station 6 done, station 7 free | Advance station 6 forward |
| 3 | Station 5 done, station 6 free | Advance station 5 forward |
| 4 | Station 4 done, station 5 free | Advance station 4 forward |
| 5 | Station 3 done, station 4 free | Advance station 3 forward |
| 6 | Station 2 done, station 3 free | Advance station 2 forward |
| 7 | Station 1 done, station 2 free | Advance station 1 forward |
| 8 | `xMasterSwitch` AND `xStartSensor` AND station 1 free | Pick new batch from Start |

> This reverse-priority order (end-first) is intentional to prevent pipeline deadlocks. Later stations always get preference, ensuring the line drains before new material is loaded.

---

## 8. Hoist Control Logic

### Full Cycle for One Load

```
IDLE
 |
 +-> [State 10] Move to pickup position
 |
 +-> [State 20] Lower hoist
 |
 +-> [State 25] Grip confirmation (one-scan hold)
 |
 +-> [State 30] Lift with load
 |
 +-> [State 40] Move to drop-off position
 |
 +-> [State 45] Lower and release
 |
 +-> [State 50] Lift empty
 |
IDLE
```

### Timer Behavior

Each station uses an independent `TON` timer:

- **Starts** when `GVL.xStationOccupied[n]` goes TRUE
- **Resets** explicitly in State 30 when the load is lifted (sets `IN := FALSE`)
- **Done flag** (`xStationDone[n]`) mirrors `timerStation[n].Q`

This ensures a timer cannot re-trigger immediately after a load is placed back into the same station during a failed lift attempt.

### Occupancy Rules

| Event | Occupancy Change |
|---|---|
| Load lifted from station N | `xStationOccupied[N] := FALSE` |
| Load placed at station N | `xStationOccupied[N] := TRUE` |
| Load placed at position 8 | `xEndSensor := TRUE` (triggers done counter) |
| Load picked from Start (pos 0) | No occupancy array entry for pos 0 |

---

## 9. Cabling and Wiring Documentation

The Excel workbook **`PRJ-26001_Galvanizing-System_PLC-Cabling-Wiring.xlsx`** contains seven sheets:

| Sheet | Contents |
|---|---|
| `Config` | Central variable definitions - project name, PLC model, voltages, I/O map. All other sheets reference this via cell links. |
| `Cover` | Project cover page, document index, and general installation notes |
| `Cable Schedule` | Full cable listing: cable number, source, destination, type, cores, size, voltage rating, shield, length, conduit/tray, status |
| `IO List` | PLC I/O assignments: tag, description, cable/wire number, connector, CODESYS address, signal type, wire colors, terminal |
| `Terminal Block Layout` | DIN rail terminal assignments: terminal number, type, wire color, cross-section, cable number, ferrule, jumper notes |
| `Wiring Legend` | IEC 60446 wire color codes, cable type specifications, abbreviations, and standards references |
| `Cable Drum Register` | Drum tracking: drum ID, cable type, full length, issued length, remaining length, location, supplier |

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
- Signal cables (4-20 mA) must be segregated from power cables - **minimum 200 mm separation**
- All cable screens/shields grounded at **one end only** (control panel end) unless noted otherwise
- Minimum cable bending radius: **6x OD** (multi-core), **8x OD** (armoured)
- All field terminations use **bootlace ferrules** (Weidmuller crimping tool or equivalent)
- Motor cables must be shielded (VFD-duty) where inverter drives are installed

---

## 10. I/O Channel Map

### WAGO CC100 On-board I/O Connector Map

| Connector | Channels | Type | CODESYS Prefix | Notes |
|:---------:|---|---|:---:|---|
| X11 | DI.1 to DI.8 | Digital Input | `%IX0.0 to IX0.7` | 24 VDC sinking, 2.8 mA, RC filter 5 us |
| X5 | DIO.1 to DIO.8 | Configurable DIO | `%IX6.x / QX` | Configurable as DI or DO |
| X6 | AI.1-AI.2 + AIO.1-AIO.2 | Analog | `%IW / %QW` | 4-20 mA or 0-10 V |
| X13 | NI/PT.1 to NI/PT.2 | Temperature | `%IW` | NI1000 / PT1000 |

### Current I/O Assignments (from IO List sheet)

| Tag | Connector | CODESYS Addr | Signal Type | Description |
|---|:---------:|:---:|:---:|---|
| X12:2 | X12 | n/a | DC 0V | PLC 0V reference |
| X12:3 | X12 | `%IX6.0` | DI 24VDC | Control button 24V input |
| 24VIN-1 | Bus | n/a | 24V DC | 24V DC Bus Input 1 |
| 24VIN-2 | Bus | n/a | 24V DC | 24V DC Bus Input 2 |
| 24VIN-3 | Bus | n/a | 24V DC | 24V DC Bus Input 3 |

> The full I/O assignment table is maintained in the `IO List` tab of the Excel workbook. CODESYS variable addresses in `GVL` must be kept synchronized with this sheet at all times.

---

## 11. Safety and Emergency Stop

### Emergency Stop Behavior

When `GVL.xEmergencyStop = TRUE`:

| Output | Forced Value | Effect |
|---|:---:|---|
| `GVL.xLowerHoist` | `FALSE` | Hoist stops descending immediately |
| `GVL.xGrabLoad` | `FALSE` | Gripper deactivated |
| `GVL.iTargetPosition` | frozen | No horizontal motion commanded |
| `iSystemState` | unchanged | State machine resumes on E-stop release |

> **Important:** After an E-stop, the operator must manually verify the hoist position and load status before releasing the stop. If a full sequence reset is required, `iSystemState := 0` can be uncommented in the E-stop branch of `PLC_PRG`.

### Hardwired Safety (Main_LD)

The `Main_LD()` Ladder Diagram sub-program is called at the end of every scan and is reserved for hardwired interlock logic that operates independently of the ST state machine. This may include:

- Physical E-stop coil monitoring
- Drive fault relay contacts
- Door/guard interlocks
- Overload contacts

---

## 12. Simulation Mode

Section 4 of `PLC_PRG` contains simulation logic for offline testing without physical I/O connected. This section must be **removed or conditionally disabled** before deploying to real hardware.

### Simulation Behavior

| Simulated Signal | Behavior |
|---|---|
| Horizontal position | `iActualPosition` increments/decrements by 1 per scan cycle toward `iTargetPosition` |
| Vertical (lowered) | `xIsFullyLowered := xLowerHoist` (instant) |
| Vertical (lifted) | `xIsFullyLifted := NOT xLowerHoist` (instant) |

### Removing Simulation for Live I/O

1. Delete or comment out the entire **Section 4** block in `PLC_PRG`
2. Map `GVL.iActualPosition` to an encoder or position feedback device
3. Map `GVL.xIsFullyLowered` and `GVL.xIsFullyLifted` to physical limit switches
4. Verify all `GVL` output variables (`xLowerHoist`, `xGrabLoad`, `iTargetPosition`) are mapped to actual PLC output channels in the CODESYS device configuration

---

## 13. Visualization and HMI

### CODESYS Visualization

The project includes a CODESYS web/panel visualization connected to the following variables:

| UI Element | Variable | Function |
|---|---|---|
| Station indicators | `GVL.xStationOccupied[1..7]` | Show occupied/free status |
| Hoist position | `GVL.iActualPosition` | Numeric display / graphic |
| Active loads counter | `iActiveLoads` | Count of loads currently in system |
| Completed loads | `iTotalDone` | Running total of finished batches |
| Reset counter button | `xResetDoneCounter` | Resets `iTotalDone` to 0 |
| Emergency stop | `GVL.xEmergencyStop` | E-stop indicator / button |
| Master switch | `GVL.xMasterSwitch` | Production enable toggle |

### HMI Panel (ePlan - HMI/PCW1)

The ePlan HMI section documents:

- **Page 1:** Main Operator Touch Panel wiring and signals
- **Page 2:** Touch panel 24V DC system power wiring

The physical HMI communicates with the CC100 via Ethernet CODESYS WebVisu over the WAGO 852-1812 switch network.

---

## 14. Standards and Compliance

| Standard | Application |
|---|---|
| IEC 61131-3 | PLC programming languages (ST, LD, FBD, SFC, IL) |
| IEC 60204-1 | Safety of machinery - electrical equipment |
| IEC 60446 | Wire identification by color codes |
| IEC 60529 | Enclosure IP rating classification |
| AS/NZS 3000 | Wiring rules (supplementary local reference) |

---

## 15. Project File Structure

```
PRJ-26001_Galvanizing_System/
|
+-- CODESYS/
|   +-- 26001_Galvanizing_System.project    Main CODESYS project file
|   +-- GVL.gvl                             Global Variable List
|   +-- PLC_PRG.st                          Main ST program
|   +-- Main_LD.ladder                      Ladder Diagram sub-program
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
```

---

## 16. Development Notes

### Known Limitations / TODO

- **Station soak times** are hardcoded at `T#10S` for all stations. Real galvanizing processes require different dwell times per bath (zinc bath typically 4-8 minutes). Parametrize via HMI-writable `TIME` variables.
- **Single-hoist only.** The state machine is designed for one hoist. Multi-hoist operation would require a significant redesign with collision avoidance logic.
- **Simulation is instant vertically.** Real hoist vertical travel takes time; replace with timed simulation (`TON`) or real limit switch feedback before plant trials.
- **`GVL.xEndSensor`** is written by the PLC program itself (set in State 45 when position = 8). On real hardware this should be an actual field sensor, and the write from `PLC_PRG` should be removed.
- **`Main_LD()`** is called but may be empty. Populate with interlock logic before commissioning.

### Commissioning Checklist

- [ ] Remove simulation section (Section 4) from `PLC_PRG`
- [ ] Map all GVL variables to physical I/O in CODESYS device tree
- [ ] Verify terminal block wiring against `Terminal Block Layout` sheet
- [ ] Update `Cable Schedule` sheet with all final cable lengths and routes
- [ ] Confirm E-stop circuit wiring per ePlan PWR schematics
- [ ] Test `Main_LD` interlock logic independently before enabling `xMasterSwitch`
- [ ] Set production soak times (`PT` on `timerStation[]`) per process engineer spec
- [ ] Validate HMI connectivity (WebVisu) over WAGO switch
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

*Last updated: 2026 | Rev A - Issued for Construction by Attila Peter Szucs*
