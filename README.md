# PLC_Eplan - TwinCAT 3 Automation Project

Beckhoff TwinCAT 3 automation and control project engineered to match the EPLAN Electric P8 schematics (`=Z01`).

---

## 🏗️ System Overview

The project controls an automated machine handling station consisting of:
- **Main Control Cabinet (`+MP`)**: Main control bus, pneumatic maintenance unit monitoring, emergency stop circuits, temperature and pressure sensing, and encoder feedback.
- **Laser Marking Station (`+LS`)**: Handshake interface with CW Laser Controller (`Lready`, `Jobloaded`, `MarkInProgress`, `Ltrig`, `STO`).
- **Decentralized Field Boxes (`+MS`)**: IP67 EtherCAT I/O box modules driving pneumatic cylinders (Infeed, Outfeed, Ejector, Clamp, Diverter, Air Blow, Bypass) and monitoring position sensors, light barriers, and servo positions.
- **Motion System**: Beckhoff `AX5206` multi-axis servo drive controlling `AM8033` synchronous servomotors (`+MS-11M1` Infeed servo, `+MS-11M2` Outfeed servo).

---

## 🔌 Hardware Architecture & Topology

```
                  ┌──────────────────────┐
                  │    Beckhoff C6015    │
                  │   IPC / Controller   │
                  └──────────┬───────────┘
                             │ EtherCAT
                  ┌──────────▼───────────┐
                  │   Beckhoff CU1128    │
                  │ 8-Port Junction (+MP)│
                  └────┬──────┬──────┬───┘
       EtherCAT Port 1 │      │      │ EtherCAT Port 3
    ┌──────────────────┘      │      └──────────────────────────────────┐
    │                         │ EtherCAT Port 2                         │
┌───▼─────────────────┐   ┌───▼─────────────────┐   ┌───────────────────▼───────────────────┐
│ +MP-6K1 (EK1100)    │   │ +LS-7K1 (EK1100)    │   │ Decentralized Field Boxes (+MS)       │
│ Main Cabinet Rack   │   │ Laser Station       │   │  ├─ +MS-9K1 (EP1018-0001, 8DI Box 1)  │
│  ├─ EL1809 (16DI)   │   │  ├─ EL1809 (16DI)   │   │  ├─ +MS-9K2 (EP1018-0001, 8DI Box 2)  │
│  ├─ EL2809 (16DO)   │   │  └─ EL2809 (16DO)   │   │  ├─ +MS-9K3 (EP1018-0001, 8DI Box 3)  │
│  ├─ EL3102 (2AI)    │   └─────────────────────┘   │  ├─ +MS-10K1 (EP2028-0001, 8DO Box 1) │
│  ├─ EL3204 (4RTD)   │                             │  └─ +MS-10K2 (EP2028-0001, 8DO Box 2) │
│  └─ EL5101 (1ENC)   │                             └───────────────────────────────────────┘
└─────────────────────┘
```

---

## 💻 Software Architecture

The PLC software implements a decoupled three-tier architecture:

1. **`GVL_IO` (Hardware Abstraction Layer)**:
   - Holds all physical PLC input (`AT %I*`) and output (`AT %Q*`) variables.
   - Annotations directly mirror EPLAN component designations (`+MS-12B1`, `+BP1-8S1`, `+MS-12Q1`, `+MS-9K1..3`, `+MS-10K1..2`).

2. **`PRG_Mapping` (Transfer / Buffer Routine)**:
   - Synchronizes raw hardware I/O values with internal process variables at the start and end of every cycle.
   - Prevents race conditions and isolates machine application logic from field wiring changes.

3. **`GVL_Inputs` & `GVL_Outputs` (Application Image)**:
   - Clean, standardized boolean and integer variables used by machine sequence controllers and state machines.

4. **`MAIN` (Execution Entry Point)**:
   - Assigned to `PlcTask` (10 ms cycle time, priority 20).
   - Coordinates `PRG_Mapping` and application logic.

---

## 📋 I/O Signal Map

### Digital Inputs (`GVL_IO` / `GVL_Inputs`)

| Device / Module | Channel | Signal Name | EPLAN Cross-Reference | Function Description |
| :--- | :--- | :--- | :--- | :--- |
| `+MP-6K10 (EL1809)` | Ch 1 | `bAirServiceUnit_PressureOK` | Sheet 12.3 (`+MS-12B1`) | Air service unit pressure switch |
| `+MP-6K10 (EL1809)` | Ch 2 | `bEStop_Pressed` | Sheet 8.7 (`+BP1-8S1:33-34`) | E-Stop button actuated (NO contact) |
| `+LS-7K10 (EL1809)` | Ch 1 | `bCW_Ready` | Sheet 7.a (`E50.0`) | CW Laser ready status |
| `+LS-7K10 (EL1809)` | Ch 2 | `bCW_AckCoded` | Sheet 7.a (`E50.1`) | CW Laser job loaded / acknowledge |
| `+LS-7K10 (EL1809)` | Ch 3 | `bCW_MarkInProgress` | Sheet 7.a (`E50.2`) | CW Laser marking in progress |
| `+MS-9K1 (EP1018)` | Ch 1 | `bCyl_Ejector_Ext` | Sheet 17.1 (`E1050.0`) | Ejector cylinder work position (AS) |
| `+MS-9K1 (EP1018)` | Ch 2 | `bCyl_Ejector_Ret` | Sheet 17.2 (`E1050.1`) | Ejector cylinder home position (GS) |
| `+MS-9K1 (EP1018)` | Ch 3 | `bCyl_Clamp_Ext` | Sheet 18.1 (`E1050.2`) | Clamp cylinder work position (AS) |
| `+MS-9K1 (EP1018)` | Ch 4 | `bCyl_Clamp_Ret` | Sheet 18.2 (`E1050.3`) | Clamp cylinder home position (GS) |
| `+MS-9K1 (EP1018)` | Ch 5 | `bCyl_Diverter_Ext` | Sheet 19.1 (`E1050.4`) | Diverter cylinder work position (AS) |
| `+MS-9K1 (EP1018)` | Ch 6 | `bCyl_Diverter_Ret` | Sheet 19.2 (`E1050.5`) | Diverter cylinder home position (GS) |
| `+MS-9K2 (EP1018)` | Ch 1 | `bCyl_Infeed_Ext` | Sheet 14.1 (`E1051.0`) | Infeed cylinder work position (AS) |
| `+MS-9K2 (EP1018)` | Ch 2 | `bCyl_Infeed_Ret` | Sheet 14.2 (`E1051.1`) | Infeed cylinder home position (GS) |
| `+MS-9K2 (EP1018)` | Ch 3 | `bCyl_Outfeed_Ext` | Sheet 16.1 (`E1051.2`) | Outfeed cylinder work position (AS) |
| `+MS-9K2 (EP1018)` | Ch 4 | `bCyl_Outfeed_Ret` | Sheet 16.2 (`E1051.3`) | Outfeed cylinder home position (GS) |
| `+MS-9K3 (EP1018)` | Ch 1 | `bLightBarrier_Infeed` | Sheet 13.1 (`E1052.0`) | Infeed light barrier |
| `+MS-9K3 (EP1018)` | Ch 2 | `bServo_Infeed_Locked` | Sheet 13.2 (`E1052.1`) | Infeed servo mechanical lock engaged |
| `+MS-9K3 (EP1018)` | Ch 3 | `bServo_Infeed_AS` | Sheet 13.4 (`E1052.2`) | Infeed servo work position |
| `+MS-9K3 (EP1018)` | Ch 4 | `bServo_Infeed_GS` | Sheet 13.5 (`E1052.3`) | Infeed servo home position |
| `+MS-9K3 (EP1018)` | Ch 5 | `bLightBarrier_Outfeed` | Sheet 15.1 (`E1052.4`) | Outfeed light barrier |
| `+MS-9K3 (EP1018)` | Ch 6 | `bServo_Outfeed_Locked` | Sheet 15.2 (`E1052.5`) | Outfeed servo mechanical lock engaged |
| `+MS-9K3 (EP1018)` | Ch 7 | `bServo_Outfeed_AS` | Sheet 15.4 (`E1052.6`) | Outfeed servo work position |
| `+MS-9K3 (EP1018)` | Ch 8 | `bServo_Outfeed_GS` | Sheet 15.5 (`E1052.7`) | Outfeed servo home position |

### Analog & Special Inputs

| Device / Module | Channel | Signal Name | Data Type | Function Description |
| :--- | :--- | :--- | :--- | :--- |
| `EL3102` | Ch 1 | `nAnalog_Pressure_Sensor` | `INT` | 0–10V / 4–20mA system pressure value |
| `EL3204` | Ch 1 | `nTemp_Heater_Band` | `INT` | PT100 temperature value |
| `EL5101` | Ch 1 | `nEncoder_Position` | `UDINT` | Incremental encoder position count |
| `EL5101` | Status | `bEncoder_Latch` | `BOOL` | Encoder latch state |

### Digital Outputs (`GVL_IO` / `GVL_Outputs`)

| Device / Module | Channel | Signal Name | EPLAN Cross-Reference | Function Description |
| :--- | :--- | :--- | :--- | :--- |
| `+MP-6K20 (EL2809)` | Ch 1 | `bAirServiceUnit_Vent` | Sheet 6.a (`+MS-12Q1`) | Air service unit vent/exhaust valve |
| `+MP-6K20 (EL2809)` | Ch 2 | `bLaserEnable` | Sheet 6.a / 8.9 | Laser enable relay |
| `+LS-7K20 (EL2809)` | Ch 1 | `bCW_DWTrig` | Sheet 7.a (`A50.0`) | CW Laser trigger signal |
| `+MS-10K1 (EP2028)` | Ch 1 | `bValve_Ejector_Ext` | Sheet 17.6 (`A1050.0`) | Ejector extend valve solenoid |
| `+MS-10K1 (EP2028)` | Ch 2 | `bValve_Ejector_Ret` | Sheet 17.7 (`A1050.1`) | Ejector retract valve solenoid |
| `+MS-10K1 (EP2028)` | Ch 3 | `bValve_Clamp_Ext` | Sheet 18.6 (`A1050.2`) | Clamp extend valve solenoid |
| `+MS-10K1 (EP2028)` | Ch 4 | `bValve_Clamp_Ret` | Sheet 18.7 (`A1050.3`) | Clamp retract valve solenoid |
| `+MS-10K1 (EP2028)` | Ch 5 | `bValve_Diverter_Ext` | Sheet 19.6 (`A1050.4`) | Diverter extend valve solenoid |
| `+MS-10K1 (EP2028)` | Ch 6 | `bValve_Diverter_Ret` | Sheet 19.7 (`A1050.5`) | Diverter retract valve solenoid |
| `+MS-10K2 (EP2028)` | Ch 1 | `bValve_Infeed_Ext` | Sheet 14.6 (`A1051.0`) | Infeed extend valve solenoid |
| `+MS-10K2 (EP2028)` | Ch 2 | `bValve_Infeed_Ret` | Sheet 14.7 (`A1051.1`) | Infeed retract valve solenoid |
| `+MS-10K2 (EP2028)` | Ch 3 | `bValve_Outfeed_Ext` | Sheet 16.6 (`A1051.2`) | Outfeed extend valve solenoid |
| `+MS-10K2 (EP2028)` | Ch 4 | `bValve_Outfeed_Ret` | Sheet 16.7 (`A1051.3`) | Outfeed retract valve solenoid |
| `+MS-10K2 (EP2028)` | Ch 5 | `bValve_AirBlow_AS` | Sheet 20.6 (`A1051.4`) | Air blow valve (Work position) |
| `+MS-10K2 (EP2028)` | Ch 6 | `bValve_AirBlow_GS` | Sheet 20.7 (`A1051.5`) | Air blow valve (Home position) |
| `+MS-10K2 (EP2028)` | Ch 7 | `bValve_Bypass_AS` | Sheet 21.6 (`A1051.6`) | Ejector bypass valve (Work position) |
| `+MS-10K2 (EP2028)` | Ch 8 | `bValve_Bypass_GS` | Sheet 21.7 (`A1051.7`) | Ejector bypass valve (Home position) |

---

## 🚀 Getting Started

### Prerequisites
- **TwinCAT 3.1** (Build 4024 or Build 4026 recommended) / **TcXaeShell** or Visual Studio with TwinCAT 3 integration.

### Opening the Project
1. Open `Eplan Project2.sln` in TwinCAT XAE (TcXaeShell).
2. Set the target system (Choose physical ARM target `TwinCAT OS (ARMV7-A)` or local test system `TwinCAT RT (x64)`).
3. Build the PLC project (`Build -> Build Solution`).
4. In the TwinCAT System Manager, complete any pending I/O links under **Untitled1 Instance -> PlcTask Inputs/Outputs**.
5. Activate Configuration (`Ctrl + Shift + F5`) and restart TwinCAT in Run Mode.

---

## 📄 License
Internal automation engineering project.
