# GaN FET PMSM Motor Control Test Bench — PCB Hardware

A two-board, 3-phase **GaN-based inverter** for driving a **PMSM / BLDC motor** with Field-Oriented Control (FOC).
The design takes inspiration from Infineon's CoolGaN™ motor-drive evaluation boards (EVAL-MTR-48V150A-GaN), whose user manuals are kept in this folder for reference.

| Board | KiCad project | Purpose |
|---|---|---|
| **Power board** | [`KicadPower/MotorDrive_Power`](KicadPower/MotorDrive_Power.kicad_pro) | 3-phase GaN half-bridges, gate drivers, current / voltage / temperature sensing, DC-link capacitors |
| **Control board** | [`KicadControlAURIX/MotorDrive_Control_AURIX`](KicadControlAURIX/MotorDrive_Control_AURIX.kicad_pro) | Carrier for the Infineon **AURIX™ TC4D7 Lite kit (A3G)**, logic supply, user I/O, 40-pin link to the power board |

---

## 1. System overview

```mermaid
flowchart LR
    DC[("DC supply<br/>≤ 48 V nominal")] -->|+Vin| PWR

    subgraph CTRL["🧠 Control board — AURIX carrier"]
        direction TB
        JACK["Power jack"] --> BUCK["MIC28515<br/>buck"] --> LDO1["LDK220C<br/>LDO"]
        MCU["AURIX TC4D7<br/>Lite kit"]
        UI["LEDs · Button · POT<br/>UART · 5-pin connectors"]
        UI --- MCU
        LDO1 --> MCU
    end

    subgraph PWR["⚡ Power board — 6-layer"]
        direction TB
        GD["Gate drivers<br/>6× 1EDN7116G"]
        PS["GaN power stage<br/>24× IGB110S101"]
        SNS["Sensing<br/>3× TLE4972 + Vph"]
        TMP["Temp sensors<br/>3× TMP1075"]
        CAP["Bulk caps<br/>DC link"]
        GD --> PS --> SNS
        CAP --- PS
    end

    MCU <==>|"40-pin header<br/>(P_HS/P_LS, IP, VP, OCD, TEMP)"| PWR
    SNS ==>|"Phase R / S / T"| M(("PMSM<br/>motor"))
```

---

## 2. Power board

### 2.1 Schematic hierarchy

The root sheet `MotorDrive_Power.kicad_sch` reuses one sheet per function and instantiates it **three times — once per phase**.

```mermaid
flowchart TD
    ROOT["MotorDrive_Power.kicad_sch<br/>(root: input connectors, TVS SMBJ48A, LED)"]
    ROOT --> CON["Connector<br/>Header 20×2 · LDK220C-R"]
    ROOT --> BULK["Bulk_Caps<br/>6× 100 µF 50 V · 2× 47 µF 50 V"]

    ROOT --> GD1["GateDrive"] & GD2["GateDrive_1"] & GD3["GateDrive_2"]
    ROOT --> PS1["PowerStage"] & PS2["PowerStage_1"] & PS3["PowerStage_2"]
    ROOT --> SN1["Sensing"] & SN2["Sensing_1"] & SN3["Sensing_2"]
    ROOT --> TS1["TempSensor"] & TS2["TempSensor_1"] & TS3["TempSensor_2"]

    classDef ph fill:#e8f1ff,stroke:#3b6fd8,color:#000
    classDef shared fill:#fff4e0,stroke:#d08a1a,color:#000
    class GD1,GD2,GD3,PS1,PS2,PS3,SN1,SN2,SN3,TS1,TS2,TS3 ph
    class CON,BULK shared
```

### 2.2 One phase leg

Each phase is a half-bridge with **4 paralleled GaN HEMTs on the high side and 4 on the low side** (8 × IGB110S101 per phase, 24 in total). Each switch group has its own single-channel gate driver.

```mermaid
flowchart LR
    MCU_HS["HS_MCU<br/>(PWM)"] --> DRV_HS["1EDN7116G<br/>high-side driver"]
    MCU_LS["LS_MCU<br/>(PWM)"] --> DRV_LS["1EDN7116G<br/>low-side driver"]

    BOOT["SS34 bootstrap<br/>+ LDK220C LDO<br/>+ 5.6 V Zener clamp"] -.supply.-> DRV_HS
    LDO["LDK220C-R LDO<br/>from +10 V"] -.supply.-> DRV_LS

    DRV_HS -->|HS gate| HS["High side<br/>4× IGB110S101 ∥"]
    DRV_LS -->|LS gate| LS["Low side<br/>4× IGB110S101 ∥"]

    VIN(["+Vin"]) --- HS
    HS --- SW(("SW node"))
    SW --- LS
    LS --- GND(["GND"])

    SW --> CS["TLE4972-AE35S5<br/>coreless current sensor"]
    CS -->|Iph_P / Iph_N| OUT1["to MCU (diff.)"]
    CS -->|OCD| OUT2["Over-current flag"]
    SW --> VDIV["Divider"] -->|Vph| OUT3["to MCU ADC"]
    CS --> PHASE(("Phase<br/>R/S/T"))
```

> The PowerStage sheet also carries a note for the alternative part **IGB070S10S1**.

### 2.3 Power tree

```mermaid
flowchart TD
    VIN["+Vin (DC bus, ≤48 V)"] --> TVS["SMBJ48A TVS"]
    VIN --> DCL["DC-link: 6×100 µF + 2×47 µF bulk<br/>+ ceramic decoupling per phase"]
    VIN --> HB["3× GaN half-bridges"]
    P10["+10 V (from control board)"] --> LDOG["LDK220C(-R) per driver"] --> GDRV["Gate-driver supply"]
    P10 --> LDOC["LDK220C-R (Connector sheet)"] --> P33S["+3V3_SNS<br/>current sensors"]
    P33["+3V3 (from control board)"] --> TMPS["TMP1075 / logic"]
```

### 2.4 Signals on the 40-pin board-to-board header

| Signal | Count | Direction (power board) | Description |
|---|---|---|---|
| `P_HS[1..3]` | 3 | in | High-side PWM per phase |
| `P_LS[1..3]` | 3 | in | Low-side PWM per phase |
| `IP_P[1..3]`, `IP_N[1..3]` | 6 | out | Differential phase-current outputs (TLE4972) |
| `VP[1..3]` | 3 | out | Phase-voltage sense |
| `OCD` | 1 | bidir | Wired over-current detect from current sensors |
| `TEMP{…}` | bus | passive | I²C bus to the three TMP1075 sensors (A0–A2 address straps) |
| `+Vin`, `+10V`, `+3V3`, `GND` | — | power | Supply rails |

Dedicated KiCad net classes exist for `P_HS`, `P_LS`, `IP_P`, `IP_N`, `VP` (with a 100 Ω tuning profile for the current-sense pairs).

### 2.5 Key components

| Function | Part | Qty |
|---|---|---|
| GaN HEMT, 100 V | Infineon **IGB110S101** (PG-VSON-4) | 24 |
| Gate driver | Infineon **1EDN7116GXTMA1** | 6 |
| Current sensor | Infineon **TLE4972-AE35S5** (coreless, with OCD) | 3 |
| Temperature sensor | TI **TMP1075DGKR** (I²C) | 3 |
| LDO | ST **LDK220C / LDK220C-R** | 7 |
| Bootstrap diode | SS34 | 3 per phase |
| Input TVS | SMBJ48A | 1 |
| Bulk capacitors | 100 µF 50 V / 47 µF 50 V | 6 / 2 |

```mermaid
pie showData title Power board — main active devices
    "GaN FETs (IGB110S101)" : 24
    "Gate drivers (1EDN7116G)" : 6
    "LDOs (LDK220C)" : 7
    "Current sensors (TLE4972)" : 3
    "Temp sensors (TMP1075)" : 3
```

### 2.6 PCB

| Parameter | Value |
|---|---|
| Size | **91 × 71 mm** |
| Copper layers | **6** (F.Cu, In1–In4, B.Cu) |
| Board thickness | ≈ 1.56 mm |
| Footprints | 343 |
| Vias / track segments / zones | 1476 / 1064 / 35 |
| Default rules | 0.2 mm track, 0.1 mm clearance, 0.6/0.3 mm via |

```text
   ┌──────────────── 6-layer stack ────────────────┐
   │ F.Cu   ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓ │  GaN FETs, drivers, power copper
   │ In1.Cu ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ │
   │ In2.Cu ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ │  inner planes / power-loop return
   │ In3.Cu ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ │
   │ In4.Cu ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ │
   │ B.Cu   ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓ │
   └────────────────────── ≈1.56 mm ───────────────┘
```

---

## 3. Control board (AURIX carrier)

```mermaid
flowchart TD
    ROOT["MotorDrive_Control_AURIX.kicad_sch"] --> PSU["PSU sheet"]
    ROOT --> CTL["Control sheet"]

    PSU --> J["Power jack"] --> T["SMBJ10A TVS"] --> B["MIC28515T-E/PHA<br/>sync. buck + inductor<br/>220 µF / 47 µF"] --> L["LDK220C-R LDO"]

    CTL --> HDR["Header 20×2<br/>→ power board"]
    CTL --> KIT["AURIX TC4D7 Lite kit pins<br/>P00 · P03 · P15 · P16 · P20–P23 · P32 · P33<br/>AN0–AN21 · /PORST · /ESR0-1"]
    CTL --> IO["4× LED · Button · POT<br/>UART TX/RX · 2× 5-pin conn · 4-pin conn<br/>3-pin SMD jumper"]
```

| Parameter | Value |
|---|---|
| Size | **145.9 × 100 mm** |
| Copper layers | 2 |
| Board thickness | ≈ 1.69 mm |
| Footprints | 114 |
| Vias / track segments / zones | 431 / 720 / 7 |

---

## 4. Repository layout

```text
MotorControl_TB_PCB/
├── README.md
├── KicadPower/                    # Power board (KiCad)
│   ├── MotorDrive_Power.kicad_pro/.kicad_sch/.kicad_pcb/.kicad_dru
│   ├── PowerStage.kicad_sch       # 8× GaN per phase
│   ├── GateDrive.kicad_sch        # 2× 1EDN7116G per phase
│   ├── Sensing.kicad_sch          # TLE4972 + phase voltage
│   ├── TempSensor.kicad_sch       # TMP1075
│   ├── Bulk_Caps.kicad_sch
│   ├── Connector.kicad_sch        # 40-pin link to control board
│   ├── MotorDrive_Power.csv       # early BOM export
│   └── Lib/                       # 1EDN7116GXTMA1 symbol + footprint
├── KicadControlAURIX/             # Control board (KiCad)
│   ├── MotorDrive_Control_AURIX.*
│   ├── PSU.kicad_sch / Control.kicad_sch
│   └── AURIX A3G TC4D7 Lite kit user guide (PDF)
├── Tags/                          # Released packages: Power_V1.0.zip, control_V1.0.zip
└── Infineon CoolGaN eval-board user manuals (PDF)
```

---

## 5. Project timeline & current status

```mermaid
timeline
    title Development history
    2026-04-02 : Initial commit
               : Migration to KiCad and fixes
    2026-04-03 : New G-series gate driver (1EDN7116G) integrated
    2026-04-05 : Control board routed
               : V1.0 sent for manufacturing
    2026-06-22 : Reference PDFs added
    2026-10-24 : Power board rev. after V1.0 (work in progress, uncommitted)
```

```mermaid
stateDiagram-v2
    direction LR
    [*] --> Schematic
    Schematic --> Layout
    Layout --> V1_0_Fab: Apr 2026
    V1_0_Fab --> Rework: Oct 2026
    Rework --> V1_1_Review: next
    V1_1_Review --> Bring_up
    Bring_up --> FOC_tests
    note right of Rework
        Current state: power board
        being revised in KiCad
    end note
```

### Status checklist

- [x] Power-board schematic (3-phase, hierarchical)
- [x] Control-board schematic (AURIX carrier + PSU)
- [x] Both PCBs laid out
- [x] **V1.0** released to manufacturing — packages in [`Tags/`](Tags)
- [ ] **Power-board revision after V1.0 (in progress, uncommitted)**
  - Sensing: 3-pin SMD jumper + 2-pin header replaced by a `Conn_01x03` with 10 Ω / 4.7 kΩ
  - GateDrive: bootstrap diodes moved to an `SS34` library symbol (3 per phase)
  - Connector / TempSensor sheets updated, PCB re-routed, net classes added
- [ ] Commit and tag the revised power board
- [ ] Board bring-up: supplies, gate drive, sensor calibration
- [ ] FOC firmware on the AURIX and motor tests

---

## 6. Tools & references

- **KiCad** (projects migrated from Altium; most parts come from the `RezaLIB` library)
- Infineon *EVAL-MTR-48V150A-GaN* user manual
- Infineon *FOC motor-drive evaluation board based on CoolGaN™ transistors* user manual
- Infineon *KIT_A3G_TC4D7_LITE* user guide (in `KicadControlAURIX/`)
