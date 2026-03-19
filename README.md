\# Galvanizing System Control Panel Design ⚡🏭



\## Project Overview

Welcome to the hardware design repository for the \*\*Galvanizing System Control Panel\*\* (Project ID: `26001\_Galvanizing\_System\_Interconnections`). 



This project details the complete electrical, logic, and network infrastructure required to control an industrial galvanizing system. The repository contains the comprehensive design documentation—ranging from high-level network topologies down to individual wire terminal mappings—developed to IEC standards. 



This design was created for \*\*Howest - Cyber3Lab\*\* to bridge the gap between physical electrical engineering and industrial automation.



\## System Architecture

The control system is modularly designed and organized into four primary functional groups:



\### 1. Power Distribution (`=PWR`)

\* \*\*Input:\*\* Standard 230V AC main power.

\* \*\*Conversion:\*\* Utilizes an industrial AC/DC converter to establish a stable 24V DC power bus.

\* \*\*Distribution:\*\* Power is safely distributed through isolated terminal blocks to separate subsystems (PLC logic, network switches, and field I/O), ensuring system stability and safety.



\### 2. Control Logic (`=PLC`)

\* The "brain" of the operation. This section details the wiring of the main Programmable Logic Controller (PLC).

\* It manages the 24V DC Digital Inputs and Outputs (I/O) that interface with the physical sensors, actuators, and safety mechanisms of the galvanizing process.



\### 3. Network Topology (`=NET`)

\* A robust industrial Ethernet network (Cat6a SF/UTP) designed for low latency and high reliability.

\* Includes uplinks to external factory networks, direct communication lines to the PLC, and dedicated service ports for maintenance and diagnostics.



\## Hardware Specifications

This design is built around highly reliable \*\*WAGO\*\* industrial components:

\* \*\*Main Controller:\*\* WAGO Compact Controller 100 (\*Part No. 751-9402\*) - Executes the galvanizing logic.

\* \*\*Power Supply:\*\* WAGO AC/DC Converter (\*Part No. 787-1202\*) - Converts 100-240V AC to 24V DC (1.3A).

\* \*\*Network Switch:\*\* WAGO 8-Port 1000Base-T Lean-Managed-Switch (\*Part No. 852-1112\*).



\## Repository Structure \& Documentation

This repository is primarily composed of EPLAN electrical schematics and supporting Excel manufacturing data. 



\### 📁 Schematics (EPLAN Exports)

Visual diagrams detailing the interconnections of the system:

\* \*\*`=PWR` Schematics:\*\* AC power entry and 24V DC distribution blocks.

\* \*\*`=PLC` Schematics:\*\* I/O card wiring and controller power.

\* \*\*`=NET` Schematics:\*\* Switch port assignments and cable routing.



\### 📁 Manufacturing Data (Excel Spreadsheets)

The build-sheets required for panel builders and field technicians to physically construct the system:

\* `IO List.csv`: The master map connecting physical hardware to the software logic (what each sensor/actuator does).

\* `Cable Schedule.csv` \& `Cable Drum Register.csv`: Exact routing, origin/destination points, lengths, and bulk ordering requirements for all wires.

\* `Terminal Block Layout.csv`: The physical arrangement of power distribution rails.

\* `Wiring Legend.csv`: Standardized color-coding and wire sizing guidelines for panel safety.



\## Getting Started

For software engineers looking to write the PLC code, start with the \*\*IO List\*\*. This will give you the exact memory addresses and variable names needed to interface with the hardware.



For panel builders and electrical engineers, begin with the \*\*EPLAN Schematics\*\* and reference the \*\*Terminal Block Layout\*\* and \*\*Cable Schedule\*\* for physical assembly.



\## Author \& Acknowledgments

\* \*\*Designed By:\*\* Howest - Cyber3Lab (Bruges, Belgium)

\* \*\*Standard:\*\* IEC 

\* \*\*Date:\*\* February 2026



\---

\*Note: This repository contains hardware design and configuration data. Logic programming files (e.g., CODESYS or structured text) may be hosted in a separate software repository depending on the project scope.\*

