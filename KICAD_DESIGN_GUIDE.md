---
title: KiCad PCB Design Guide
version: 1.0
date: 2026-11-10
scope: Shared conventions for any KiCad project (repo layout, design rules, schematic, layout, verification, fabrication output, versioning)
primary_domain: Motor controllers (BLDC/DC), MCU boards (STM32, ESP32, RP2350)
kicad_versions_covered: [4, 7, 10]
units: millimetres unless stated otherwise
keywords: [MUST, MUST NOT, SHOULD, SHOULD NOT, MAY]   # RFC 2119 meaning
sources:
  - https://github.com/PhamChinhCode/Mini-PID-X2
  - https://github.com/OLIMEX/ESP32-EVB
  - https://github.com/OpenDrone-hw/OpenESC-20x20
  - https://github.com/OpenDrone-hw/OpenFC-Lite-Mini
  - https://github.com/Source-Robotics/Spectral-Micro-BLDC-controller
  - https://github.com/ziteh/moco
  - https://github.com/LT310/bldc-hardware
---

# KiCad PCB Design Guide

## 0. How to read this document

This guide is written for both human engineers and AI agents.

- **Rule IDs.** Every rule has a stable ID (for example `LIB-02`, `PWR-05`). Cite the ID when you apply or violate a rule.
- **Requirement levels** follow RFC 2119:
  - **MUST / MUST NOT**: mandatory.
  - **SHOULD / SHOULD NOT**: strong default; deviation needs a written reason.
  - **MAY**: optional.
- **Evidence tags:**
  - `[OBSERVED]`: a fact extracted directly from the source projects' `.kicad_pro`, `.kicad_pcb`, `.kicad_dru` or `.kicad_sch` files, at their latest commit on 2026-11-10.
  - `[RECOMMENDED]`: a conclusion drawn from those observations. Not a rule of any source project.
- **Units.** All dimensions are in millimetres (mm). Copper weight is in oz (1 oz ≈ 35 µm, 2 oz ≈ 70 µm).
- **Placeholders.** `<project>` stands for the KiCad project name.
- **Machine-readable data.** Section 4.4 contains a YAML block with the design rule profiles. Parse that block instead of the prose tables when you need numbers.

### Glossary
| Term | Meaning |
|---|---|
| Net class | KiCad group of nets that share width, clearance and via rules |
| Zone | Copper pour (filled polygon) assigned to a net |
| Keepout / rule area | Region where copper, vias or tracks are forbidden |
| Net Tie | Footprint that intentionally joins two nets at one point (for example GND and GNDPWR) |
| Kelvin connection | Separate sense traces taken from the exact ends of a shunt resistor |
| Switch node | High-dv/dt node of a buck converter or half-bridge |
| Bulk capacitor | Large capacitor on the main power rail |
| ERC / DRC | Electrical Rules Check (schematic) / Design Rules Check (PCB) |
| Schematic parity | DRC check that the PCB matches the schematic netlist |
| LCSC | Part number at the LCSC / JLCPCB distributor |
| MPN | Manufacturer Part Number |
| DNP | Do Not Populate |
| CPL | Component Placement List (pick-and-place file) |

---

## 1. Source projects

| ID | Project | Board type | KiCad | Copper layers | Size | Licence |
|---|---|---|---|---|---|---|
| P1 | Mini-PID-X2 | 2× brushed-DC driver (DRV8871) + STM32F103 | 10 | 2, 1.6 mm | 55 × 40 | none (no LICENSE file) |
| P2 | OLIMEX ESP32-EVB | ESP32 board: Ethernet, CAN, relays, Li-Po | 4 → 7 (Rev L) | 2, 1.6 mm | ~75 × 75 | Apache-2.0 |
| P3 | OpenESC-20x20 | 4-in-1 BLDC ESC, 40 A/channel | 10 | 6, 1.6 mm, 2 oz outer | 31.2 × 33 | CERN-OHL-S-2.0 |
| P4 | OpenFC-Lite-Mini | RP2354A flight controller | 10 | 6, 1.0 mm | 26.9 × 26.9 | CERN-OHL-S-2.0 |
| P5 | Spectral Micro | FOC BLDC driver, STM32F103 | n/a | n/a | 39 × 39 | GPL-3.0 |
| P6 | moco | Family of 7 BLDC/PMSM boards (DRV83xx, STSPIN32) | 7 | 2 and 4 | 32–50, square and round | CERN-OHL-P v2 (HW), MIT (FW) |
| P7 | LT310 bldc-hardware | VESC 4.12 (STM32F4 + DRV8302) | 4 (legacy) | 4 (In1 = `GND`, In2 = `GND2`) | ~42 × 65 | CC BY-SA 4.0 |

`[OBSERVED]` P5 contains **no KiCad files**, only firmware, STEP models and photos. Use P5 only as a reference for README, release and version documentation. Do not use it as a source of design rules.

---

## 2. Repository layout

### 2.1 Reference tree `[RECOMMENDED]` (based on P3/P4)

```
<board-name>/
├── README.md                 # specs, photos, connectors, firmware flashing
├── DESIGN.md                 # or AGENTS.md: architecture, power tree, key parts, layout constraints
├── LICENSE
├── CHANGELOG.md              # or VERSIONS.md: revision table (section 9)
├── hardware/
│   ├── <project>.kicad_pro
│   ├── <project>.kicad_sch   # root sheet
│   ├── power.kicad_sch, mcu.kicad_sch, ...   # one sub-sheet per function block
│   ├── <project>.kicad_pcb
│   ├── <project>.kicad_dru   # custom rules only
│   ├── sym-lib-table
│   ├── fp-lib-table
│   ├── lib.kicad_sym
│   ├── lib.pretty/
│   ├── lib.3dshapes/
│   ├── KiCad-Library/        # optional: git submodule with the shared library
│   ├── fabrication-toolkit-options.json
│   └── tools/                # board-specific scripts
├── firmware/                 # or a link to the firmware repo
├── images/                   # top/bottom 3D renders for the README
└── production/               # exported Gerber zip, BOM, CPL, ibom.html
```

### 2.2 Rules
| ID | Level | Rule |
|---|---|---|
| REPO-01 | MUST | Choose the project name once and never rename it. `[OBSERVED]` P3 keeps the name `4in1-mini` (different from the repo name) because renaming would break fab archive names and release assets. |
| REPO-02 | MUST NOT | Commit lock and cache files: `*.lck`, `~*.lck`, `fp-info-cache`, `*-backups/`. |
| REPO-03 | MAY | Commit `*.kicad_prl` (it only stores view state). |
| REPO-04 | SHOULD | Use git tags or releases for revisions. SHOULD NOT copy the whole project per revision. `[OBSERVED]` P2 keeps `HARDWARE/REV-A` … `REV-L` as separate folders; the repo is ~543 MB. |
| REPO-05 | MAY | Keep several boards in one repo under `hardware/<board>/` with a shared library one level up. `[OBSERVED]` P6 uses `hardware/moco.kicad_sym` and `hardware/moco.pretty`. |
| REPO-06 | SHOULD | When migrating from another EDA tool, keep the original files in `legacy/`. `[OBSERVED]` P1 keeps its EasyEDA Pro `.epro` there. |

---

## 3. Project setup

### 3.1 KiCad version
| ID | Level | Rule |
|---|---|---|
| VER-01 | MUST | State the KiCad version in the README (for example "Designed in KiCad 10"). |
| VER-02 | MUST | All contributors use the same KiCad major version. Files saved by a newer version cannot be opened by an older one. |

### 3.2 Libraries
| Tier | Method | Example |
|---|---|---|
| KiCad standard library | Generic R, C, LED, connectors | `[OBSERVED]` P1: 71 of 93 symbols come from `Device` |
| Project-local library | `lib.kicad_sym` + `lib.pretty` + `lib.3dshapes`, referenced with `${KIPRJMOD}` in `sym-lib-table` / `fp-lib-table` | P4 (`lib`), P1 (`MiniPID_Manual`) |
| Shared multi-board library | Pinned git submodule, path exposed through a text variable | P3/P4: `OPENDRONE_LIB = ${KIPRJMOD}/KiCad-Library` |
| Company library | Fully in-house library | P2: `OLIMEX_RCL`, `OLIMEX_IC`, … |

| ID | Level | Rule |
|---|---|---|
| LIB-01 | MUST | Reuse before drawing. Search the shared library and its parts list (`PARTS-USED.md` in P3/P4) before creating a symbol or footprint. |
| LIB-02 | SHOULD | Import new parts that have an LCSC number with `easyeda2kicad`, then verify the footprint against the datasheet. |
| LIB-03 | SHOULD | Store the exact datasheet revision used for the design, indexed by SHA-256 (`datasheet/manifest.json` in P3/P4). |
| LIB-04 | MUST | Update the shared library in its own commit, then re-run DRC. |
| LIB-05 | MAY | Freeze an old library: keep using it, add nothing to it. |
| LIB-06 | MUST | Reference project-local libraries with `${KIPRJMOD}`, never with an absolute path. |

### 3.3 Text variables (`.kicad_pro`)
`[OBSERVED]` P3/P4 store board facts as text variables and reuse them in the title block, silkscreen and docs: `DOC_NAME`, `DOC_MCU`, `DOC_INPUT`, `DOC_RATING`, `DOC_FIRMWARE`, `DOC_TAGLINE`, `DOC_VARIANT`.

| ID | Level | Rule |
|---|---|---|
| VAR-01 | SHOULD | Define at least `BOARD_NAME`, `REV` and `DATE`, and use them in both the title block and the silkscreen so they never disagree. |

### 3.4 Net classes
| ID | Level | Rule |
|---|---|---|
| NC-01 | MUST | Assign net classes with **patterns** (Board Setup → Net Classes → Assignments), not net by net. |
| NC-02 | SHOULD | For motor controllers, use the net-class set in the table below. |

`[OBSERVED]` Net classes in the source projects (track width in mm):

| Project | Net classes |
|---|---|
| P1 | `Default` 0.254 · `GND` 0.5 · `PWR_3V3` 0.4 · `PWR_5V` 0.6 · `PWR_HI` 1.0 (via 0.8/0.4) · `USB` 0.2 with 0.15 clearance |
| P6 rd501 | `Default` 0.2 · `Analog` 0.3 (shunt sense SN/SP, V_SENSE) · `Driver` 0.25 (gates GH/GL, phases) · `GND` 0.4 · `Power-1` 0.5 · `Power-2` 1.0 |
| P6 sc1 | `Default` 0.2 · `Power` 0.35 with 0.35 clearance |
| P4 | `Default` 0.1 · `50Ohm` for RF antenna traces |
| P3, P4 | Only `Default` for power: high current is carried by zones, not tracks |

`[RECOMMENDED]` Net-class set for motor controllers:

| Net class | Applies to | Notes |
|---|---|---|
| `Default` | Digital signals | Set to the fab's capability |
| `GND` | Ground | Wider than Default; prefer pours |
| `PWR_LOGIC` | 3V3, 5V | |
| `PWR_HI` | VIN, VBAT, phase outputs | Prefer pours. Size width with *PCB Calculator → Track Width* |
| `Gate` | GH_x, GL_x, SH_x | Short; routed next to its return path |
| `Analog_Sense` | Shunt Kelvin traces, voltage dividers | Larger clearance; away from switch nodes |
| `USB` / `CAN` / `RF_50Ohm` | Differential pairs, antennas | Define diff-pair or impedance settings |

---

## 4. Design rules

### 4.1 Observed rule profiles `[OBSERVED]`

| Parameter (mm) | **A. 2-layer general**<br>P1, P2 | **B. 4-layer power**<br>P6 | **C. 6-layer high-density**<br>P3, P4 |
|---|---|---|---|
| Min clearance | 0.15 – 0.18 | 0.13 – 0.20 | 0.09 – 0.10 (0.16 on 2 oz outer layers) |
| Min track width | 0.127 – 0.20 | 0.13 – 0.20 | 0.09 – 0.10 (0.16 on 2 oz outer layers) |
| Common vias (diameter/drill) | 0.6/0.3, 0.45/0.3, 0.9/0.6 | 0.56/0.3, 0.5/0.2, 0.7–0.8/0.4 | **0.35/0.20**, 0.45/0.3 |
| Min annular ring | 0.075 – 0.13 | 0.13 – 0.20 | 0.075 |
| Copper to board edge | 0.30 | 0.30 – 0.35 | 0.20 |
| Hole to hole | 0.25 – 0.30 | 0.25 – 0.50 | 0.20 |
| Min text height | 0.64 – 0.8 | 0.8 | 0.8 |
| Most-used tracks | 0.2 (signal), 0.4 – 2.0 (power) | 0.2, 0.3 – 1.0 | 0.1 – 0.2, 0.3 – 1.0 |

### 4.2 Observed stackups `[OBSERVED]`

| Board | Stackup (top to bottom) |
|---|---|
| P3 OpenESC-20x20 (1.69 mm) | F.Cu **70 µm** / prepreg 0.1 / In1 35 µm / core 0.545 / In2 35 µm / prepreg 0.1 / In3 35 µm / core 0.545 / In4 35 µm / prepreg 0.1 / B.Cu **70 µm** |
| P4 OpenFC-Lite-Mini (1.0 mm) | F.Cu 35 µm / prepreg 0.1 / In1 15 µm / core 0.16 / In2 15 µm / prepreg 0.33 / In3 15 µm / core 0.16 / In4 15 µm / prepreg 0.1 / B.Cu 35 µm |
| P6 rd501 (1.68 mm) | F.Cu **70 µm** / core 0.21 / In1 15 µm / prepreg 1.065 / In2 15 µm / core 0.21 / B.Cu **70 µm** |
| P6 sc1, rd321 (1.6 mm) | F.Cu 35 µm / prepreg 0.1 / In1 35 µm (type `power`) / core 1.24 / In2 35 µm / prepreg 0.1 / B.Cu 35 µm |
| P7 VESC 4.12 | 4 layers; inner layers renamed `GND` and `GND2` (two ground planes) |

| ID | Level | Rule |
|---|---|---|
| STK-01 | MUST | Enter the fab's exact stackup in Board Setup → Physical Stackup, so impedance and fab output are correct. |
| STK-02 | SHOULD | Use **2 oz outer copper** on boards carrying more than ~10 A (as P3 and P6 rd501 do). |
| STK-03 | MUST | When using 2 oz outer copper, raise min clearance and track width **on outer layers only**, with a custom rule (section 4.3). Do not raise the global minimum: it would constrain inner-layer routing for no reason. |
| STK-04 | SHOULD | On 4+ layer boards, make the layer directly under the component side a **solid ground plane**. |

### 4.3 Custom rules file (`.kicad_dru`)
| ID | Level | Rule |
|---|---|---|
| DRU-01 | MUST | Keep fab-capability numbers (min clearance, track, via) in `.kicad_pro` → `board.design_settings.rules`. Import them via Board Setup → *Import Settings from Another Board* from a template board. |
| DRU-02 | MUST NOT | Repeat in `.kicad_dru` any number already defined in `.kicad_pro`. Two copies of a number drift apart. |
| DRU-03 | MUST | Put shared rules above a marker line and board-specific rules below it. |
| DRU-04 | MUST | Name every rule and add a comment explaining why it exists. |

Template:
```lisp
(version 1)

(rule "silkscreen over pad"
  (constraint silk_clearance (min 0.15mm))
  (condition "A.Type == 'Pad'"))

# --- board-specific rules below this line ---

# 2 oz outer copper etches with a wider taper: raise minimums on outer layers only
(rule "2 oz outer copper, clearance"
  (constraint clearance (min 0.16mm))
  (layer outer))
(rule "2 oz outer copper, track width"
  (constraint track_width (min 0.16mm))
  (layer outer))

# [RECOMMENDED] extra clearance between battery-voltage nets and logic nets
(rule "HV to logic"
  (constraint clearance (min 0.5mm))
  (condition "A.NetClass == 'PWR_HI' && B.NetClass == 'Default'"))
```

### 4.4 Machine-readable rule profiles

```yaml
# Values in mm. Source: [OBSERVED] ranges from section 4.1, collapsed to one
# conservative value per parameter. Always check against the chosen fab's capability.
rule_profiles:
  A_2layer_general:
    layers: 2
    thickness: 1.6
    copper_oz: {outer: 1}
    min_clearance: 0.15
    min_track_width: 0.20
    default_via: {diameter: 0.6, drill: 0.3}
    power_via: {diameter: 0.8, drill: 0.4}
    min_annular_ring: 0.13
    copper_to_edge: 0.30
    hole_to_hole: 0.25
    min_text_height: 0.8
    examples: [P1, P2]
  B_4layer_power:
    layers: 4
    thickness: 1.6
    copper_oz: {outer: 1, inner: 1}   # P6 rd501 uses 2 oz outer
    min_clearance: 0.15
    min_track_width: 0.15
    default_via: {diameter: 0.56, drill: 0.3}
    power_via: {diameter: 0.8, drill: 0.4}
    min_annular_ring: 0.13
    copper_to_edge: 0.30
    hole_to_hole: 0.25
    min_text_height: 0.8
    stackup_hint: "F.Cu signal / In1 GND plane / In2 power or signal / B.Cu signal"
    examples: [P6]
  C_6layer_high_density:
    layers: 6
    thickness: 1.6          # P4 uses 1.0
    copper_oz: {outer: 1, inner: 0.5}   # P3 uses outer 2 oz, inner 1 oz
    min_clearance: 0.09
    min_track_width: 0.09
    outer_2oz_override: {min_clearance: 0.16, min_track_width: 0.16}
    default_via: {diameter: 0.35, drill: 0.20}
    min_annular_ring: 0.075
    copper_to_edge: 0.20
    hole_to_hole: 0.20
    min_text_height: 0.8
    examples: [P3, P4]
```

---

## 5. Schematic conventions

### 5.1 Hierarchy `[OBSERVED]`
| Project | Sheets |
|---|---|
| P4 | `power`, `rp2350a` (MCU), `imu`, `osd`, `blackbox`, `pads` (external solder pads) |
| P6 rd501 / od501 | `MCU`, `Motor Driver`, `Rotary Position Sensor`, `CAN BUS` |
| P6 sc1 | `Power Stage`, `Rotary Sensor` |
| P7 | `Power`, `mosfets`, `STM32F4 64LQFP`, `CAN`, `hall_filters`, `temp` |
| P3 | Root sheet (power, current sense, connector) plus `ESC.kicad_sch` **drawn once and instantiated 4 times** (ESC1–ESC4) |
| P1, P2 | Single flat sheet |

| ID | Level | Rule |
|---|---|---|
| SCH-01 | SHOULD | Use hierarchical sheets when the board has more than ~60 components. One sheet per function block. |
| SCH-02 | SHOULD | Draw repeated blocks (motor channels, half-bridges) once and instantiate the sheet. Nets then get prefixes such as `/ESC1/MotorA`, which makes pattern-based net-class assignment easy. |
| SCH-03 | SHOULD | Connect sheets with hierarchical labels. SHOULD NOT use global labels except where unavoidable. `[OBSERVED]` the source projects use almost no global labels. |

### 5.2 Power and ground naming
`[OBSERVED]` Names in use: `GND`, `+3V3`, `+3.3V`, `+5V`, `+10V`, `+BATT`, `VBUS`, `VCC`, `+1.8V_GYRO`, `+5V_USB`, `+5V_BUCK`, `GNDPWR`, `GNDA`.

| ID | Level | Rule |
|---|---|---|
| PWR-01 | MUST | Pick one style (`+3V3` or `+3.3V`) and use it in every project. |
| PWR-02 | SHOULD | Name dedicated rails by purpose, for example `+1.8V_GYRO`. `[OBSERVED]` P4 gives the IMU its own analog supply, separate from the 3.3 V logic rail. |
| PWR-03 | SHOULD | On power boards, separate power ground and signal ground (`GNDPWR`, `GND`, optionally `GNDA`) and join them at one point with a **Net Tie**. `[OBSERVED]` P6 rd501 uses NT1–NT4. |
| PWR-04 | MUST | Place `PWR_FLAG` on power nets that enter through a connector, so ERC does not report false "power input not driven" errors. |

### 5.3 Component fields (BOM data)
`[OBSERVED]` Field coverage:

| Project | Fields used | Coverage |
|---|---|---|
| P1 | `LCSC`, `MPN`, `Manufacturer`, `Datasheet`, `Description` | 89 / 93 |
| P3 | `LCSC`, `MPN`, `Manufacturer`, `Cost`, `Datasheet` | 59 / 62 |
| P4 | `LCSC`, `MPN`, `Manufacturer` | 96 / 146 |
| P6 rd501 | `JLCPCB Part #`, `MFR.Part #`, `Basic Part` | 103 / 131 |
| P6 sc1 | `LCSC`, `MFR. Part#`, `Alt.`, `Note` | 91 / 120 |
| P2 | `Fieldname 1/2/3` (meaningless names) | n/a |

Standard field set. Field names MUST be spelled exactly like this in every project so BOM scripts work:

| Field | Required | Example |
|---|---|---|
| `MPN` | yes | `DRV8871DDAR` |
| `Manufacturer` | yes | `Texas Instruments` |
| `LCSC` | yes, if assembled at JLCPCB | `C86590` |
| `Datasheet` | yes | URL or `${LIB}/datasheet/<file>.pdf` |
| `Description` | yes | `Brushed DC motor driver 45V 3.6A` |
| `Alt` | no | Alternative part number |
| `Basic Part` | no | `Yes` / `No` (JLC basic parts are cheaper) |
| `Note` | no | `DNP`, assembly note |

| ID | Level | Rule |
|---|---|---|
| BOM-01 | MUST | Fill `MPN`, `Manufacturer`, `Datasheet`, `Description` on every non-power symbol; also `LCSC` when assembling at JLCPCB. |
| BOM-02 | MUST | The schematic is the single source of truth for the BOM. MUST NOT maintain a hand-written BOM in parallel. Any CSV BOM MUST be generated from the schematic. `[OBSERVED]` P1: R4 was changed to 51 kΩ but its `LCSC` field still held the 49.9 kΩ part. |
| BOM-03 | MUST | Sync fields from schematic to PCB (*Update PCB from Schematic*) before exporting fab files. |
| BOM-04 | MUST | Put critical ratings (capacitor voltage, shunt power, inductor current) in `Value` or `Description`. |
| BOM-05 | MUST | When `Value` changes, update `MPN` and `LCSC` in the same edit. |

### 5.4 PCB-only components
`[OBSERVED]` P3 has a bulk bank of 22 × 10 µF capacitors **on the PCB with no schematic symbols**, and has to warn: "Do not run update-from-schematic without checking what it would delete."

| ID | Level | Rule |
|---|---|---|
| SCH-04 | SHOULD NOT | Place components on the PCB only. Draw them in the schematic, for example in a "Bulk decoupling" sheet. |
| SCH-05 | MUST | If PCB-only components are unavoidable, list them in DESIGN.md. |

---

## 6. Layout guidelines

### 6.1 General
| ID | Level | Rule |
|---|---|---|
| LAY-01 | SHOULD | Place parts along the power flow: input connector → protection → bulk capacitor → power stage → outputs. Keep logic away from switch nodes. |
| LAY-02 | MUST | Place decoupling capacitors next to the IC power pins, with the ground via at the capacitor pad. |
| LAY-03 | MUST | Buck converters: keep the inductor and input/output capacitors tight to the IC and pull copper back under the switch node. Follow the regulator datasheet layout section. `[OBSERVED]` P4 follows RP2350 datasheet section 6.3.8.1. |
| LAY-04 | MUST NOT | Route signals across splits in the reference ground plane (4+ layer boards). |
| LAY-05 | SHOULD | Pour ground on outer layers and stitch it with vias. `[OBSERVED]` P3 has 22 GND zones on F.Cu. |
| LAY-06 | SHOULD | Bring SWD, UART and power out to test pads with silkscreen names, for pogo-pin jigs. `[OBSERVED]` P1 labels pads on the bottom side; P3 has a pogo-pin flashing jig. |
| LAY-07 | MUST | Assign a net (usually GND) to mounting holes in the schematic. `[OBSERVED]` P1 open issue: unassigned holes H1–H4 cause 8 ERC errors, and a GND track shorts to pad H4. |

### 6.2 Motor power stage (BLDC / DC)
| ID | Level | Topic | Rule |
|---|---|---|---|
| MOT-01 | SHOULD | Phase and supply nets | Use **multi-layer zones** instead of tracks. `[OBSERVED]` P3: `MotorA` poured on all 6 layers, `+BATT` on F.Cu and B.Cu. P6 rd501: `MOTOR_A/B/C` and `Vdrive` on F and B. |
| MOT-02 | MUST | Current-carrying vias | Use many vias in parallel. `[OBSERVED]` P3 has 820 vias of 0.35/0.2. `[RECOMMENDED]` Estimate ~0.5–1 A per 0.3 mm via with 1 oz plating, then verify with KiCad PCB Calculator. |
| MOT-03 | MUST | Commutation loop | Place bulk and ceramic capacitors right at the high-side/low-side MOSFETs to **minimise the current loop area**. |
| MOT-04 | MUST | Gate drive | Keep gate traces (GH/GL) short and routed alongside their return (SH or ground). Use a `Gate` net class (`[OBSERVED]` P6: 0.25 mm). |
| MOT-05 | MUST | Shunt current sense | Route **Kelvin** sense traces (SP/SN) from the shunt pad ends as a tight pair, in an `Analog_Sense` net class. Place the RC filter at the amplifier pins. `[OBSERVED]` P3 Rev 3.2 added a matched input network (1 kΩ + 100 nF per leg) to reject high-side common-mode noise. |
| MOT-06 | SHOULD | Ground strategy | Separate `GNDPWR` and `GND` and join them at one point with a Net Tie (P6), or dedicate inner planes to ground (`[OBSERVED]` P7: `GND`, `GND2`). |
| MOT-07 | MUST | Input protection | Size TVS clamping against the MOSFET Vds margin. `[OBSERVED]` P1 uses a P-MOSFET reverse-polarity switch, SMAJ18A TVS and 100 µF electrolytic. P3 Rev 3 **removed** its TVS diodes because the rail voltage was too close to MOSFET Vds for a TVS to protect anything. |
| MOT-08 | MUST | Thermal | Use thermal reliefs on signal pads only. Connect power pads and MOSFET tabs to zones **solid**, with thermal vias under the pad. `[OBSERVED]` P3 ships `esc_thermal.py` to compute losses and temperature from MOSFET datasheet values. |
| MOT-09 | SHOULD | Supply voltage sense | Use a divider plus filter capacitor into the ADC. `[OBSERVED]` P4: 100 kΩ / 10 kΩ + 100 nF. |

### 6.3 MCU, RF and sensors
| ID | Level | Rule |
|---|---|---|
| RF-01 | MUST | Place Wi-Fi module antennas at the board edge, with the antenna overhanging or over a **keepout** with no copper on any layer. `[OBSERVED]` P2 has 8 keepout areas. |
| RF-02 | MUST | Route RF traces in a dedicated impedance net class (`[OBSERVED]` P4: `50Ohm`), with width calculated from the stackup. |
| SEN-01 | SHOULD | Give IMUs and analog sensors their own supply rail. `[OBSERVED]` P4 uses a dedicated 1.8 V LDO for the gyro. |
| SEN-02 | SHOULD | Design multi-source footprints so parts can be swapped without a layout change. `[OBSERVED]` P4's LGA-14 land pattern accepts Bosch BMI2xx, TDK ICM-4xxxx and ST LSM6D. Apply the same idea to MOSFETs (standard 3×3 DFN) and gate drivers. |
| SEN-03 | SHOULD | Give encoder inputs pull-ups and RC filters. `[OBSERVED]` P1: 10 kΩ pull-up, 1 kΩ + 1 nF (fc ≈ 160 kHz). |
| SEN-04 | MUST | Check logic-level compatibility when a 5 V encoder feeds a 3.3 V MCU. |

### 6.4 Silkscreen
| ID | Level | Rule |
|---|---|---|
| SILK-01 | MUST | Print board name, revision, date, supply polarity and connector pin names. `[OBSERVED]` P5: "Spectral motor controller micro v2.0, 10.1.2024". |
| SILK-02 | MUST | Spell-check silkscreen text. `[OBSERVED]` P1 has "Brigde" instead of "Bridge". |
| SILK-03 | SHOULD | Avoid words that may trigger export restrictions. `[OBSERVED]` P3 Rev 3.3 changed "OpenDrone" to "incutec" on the silkscreen for this reason. |

---

## 7. Verification (ERC / DRC)

Commands used by the KiCad 10 projects (P1, P3, P4):

```bash
kicad-cli sch erc <project>.kicad_sch
```

```bash
kicad-cli pcb drc --schematic-parity --refill-zones <project>.kicad_pcb
```

```bash
kicad-cli sch export netlist --format kicadsexpr -o <project>.net <project>.kicad_sch
```

| ID | Level | Rule |
|---|---|---|
| CHK-01 | MUST | Run ERC and DRC before every commit, pull request and fab order, with `--schematic-parity` to catch PCB/schematic mismatches. |
| CHK-02 | MUST | Existing approved violations may remain, but **a new violation type or a higher count requires review** (P3/P4 policy). |
| CHK-03 | SHOULD NOT | Set rule severities to `ignore`, except metadata rules (`footprint_type_mismatch`, `footprint_filters_mismatch`, `missing_courtyard` for legacy footprints). For physical rules, add a per-violation *exclusion* with a reason instead. `[OBSERVED]` P1 ignores 5 DRC rules (metadata). P4 ignores 17, including `copper_edge_clearance`, `solder_mask_bridge`, `courtyards_overlap` and `via_dangling`, which can hide real defects. |
| CHK-04 | MUST | Close KiCad before any script writes to a KiCad file. KiCad caches library tables and overwrites files on save. `[OBSERVED]` P1 tools refuse to run while a `~*.lck` file exists. |
| CHK-05 | MUST NOT | Edit `.kicad_sch`, `.kicad_pcb` or `.kicad_dru` as text. Use KiCad, the `pcbnew` Python API (KiCad's bundled Python) or `kicad-skip`. `.kicad_pro` is JSON and MAY be edited directly for metadata. |
| CHK-06 | MUST | AI agents and scripts MAY write metadata (MPN, Manufacturer, LCSC, Cost, Datasheet, text variables) but MUST NOT change nets, wiring, routing, placement, footprint assignment or any value that changes the circuit without human approval (P3/P4 policy). |

---

## 8. Fabrication output

| Output | Tools used in the source projects | Notes |
|---|---|---|
| Gerber + drill zip | **Fabrication Toolkit** plugin (P3/P4, configured in `fabrication-toolkit-options.json`); `kicad-cli`; P1 `fab` script | Set `ARCHIVE_NAME` to the revision (for example `OpenESC-20x20-rev3.3`). Enable `AUTO FILL` to refill zones before export. |
| BOM + CPL for JLCPCB | Fabrication Toolkit; P1 `fab` | Origin at the board's bottom-left corner. Fields `FT Rotation Offset` / `FT Position Offset` correct part orientation. |
| Interactive BOM | InteractiveHtmlBom plugin (P1 `bom/ibom.html`) | Hand-assembly aid |
| Per-fab Gerber sets | P1 ships zips for JLCPCB, PCBWay, Elecrow, FusionPCB | File-naming conventions differ by fab |
| Schematic PDF | P2, P7 (`BLDC_4.pdf`) | Readable without KiCad |
| 3D render / STEP | `kicad-cli pcb render`, `kicad-cli pcb export step` | Render for the README; STEP for enclosure design (P5, P2 `BOX-DESIGN`) |

Reference pipeline (P1 `tools/minipid`):
```bash
minipid sync && minipid refill && minipid check && minipid fab
```
1. `sync`: copy Value, LCSC, MPN from schematic to PCB.
2. `refill`: refill all zones.
3. `check`: ERC, DRC, parity; exits with code 1 on errors.
4. `fab`: export Gerber, drill, BOM, CPL.

| ID | Level | Rule |
|---|---|---|
| FAB-01 | MUST | Re-export fab files from the current PCB right before ordering. MUST NOT reuse a zip already stored in the repo. |
| FAB-02 | SHOULD | Automate the pipeline (sync → refill → check → export) in one script or CI job. |

---

## 9. Versioning and documentation

### 9.1 Revisions
| ID | Level | Rule |
|---|---|---|
| REV-01 | MUST | Keep a **revision table**: revision, date, change, export name. `[OBSERVED]` P3 documents v0.1 to Rev 3.3 with the reason for each change (TVS removal, CSA filter, silkscreen change). |
| REV-02 | SHOULD | Mark validated builds ("Validated build", quantity produced). |
| REV-03 | MUST | Create a git tag or release for each fab order, with Gerber, BOM, CPL and PDF attached. |
| REV-04 | SHOULD | When the board ships with firmware, keep a hardware ↔ firmware ↔ bootloader compatibility table. `[OBSERVED]` P5 `VERSIONS.md`. |
| REV-05 | MUST | The revision printed on the silkscreen matches the git tag. |

### 9.2 README minimum content
1. One-sentence description and top/bottom renders.
2. **Spec table:** input voltage, continuous current, MCU, interfaces, dimensions, layer count, thickness, copper weight.
3. Connector and pinout table.
4. Firmware flashing and update procedure.
5. Folder structure.
6. Known issues / open tasks (P1 has this section).
7. Licence.

### 9.3 DESIGN.md (or AGENTS.md) template
Based on P3/P4:
- **Architecture:** block diagram.
- **Power:** ASCII power tree, for example:
  ```
  Battery ─► shunt ─► +BATT ─┬─► MOSFETs
                             └─► Buck ─► +10V ─► LDO ─► +3V3
  ```
- **Key parts:** table of Ref, Part, LCSC, note (is a substitute allowed?).
- **Connectors and I/O,** GPIO map.
- **Layout rules:** constraints that must never be broken, for example "do not merge the 1.8 V analog rail into 3.3 V".
- **Repo:** which file is what, KiCad version, design rules file, fab config.
- **Rules:** team workflow, for example "only one person edits the layout at a time", because KiCad files do not merge.

### 9.4 Hardware licences
| Licence | Used by | Meaning |
|---|---|---|
| CERN-OHL-P v2 | P6 | Permissive; commercial use allowed, no obligation to publish |
| CERN-OHL-S v2 | P3, P4 | Strong copyleft; derivatives must publish their design |
| CC BY-SA 4.0 | P7 | Share-alike copyleft |
| Apache-2.0 | P2 | Permissive |
| GPL-3.0 | P5 | Copyleft (designed for software) |

| ID | Level | Rule |
|---|---|---|
| LIC-01 | MUST | Every repo has a LICENSE file. |
| LIC-02 | SHOULD | Use a CERN-OHL variant (P, W or S) for hardware and a separate licence for firmware (MIT, BSD or GPL). |
| LIC-03 | MAY | Apply for OSHWA certification (as P3/P4 did). |

---

## 10. Pre-order checklist

**Schematic**
- [ ] ERC clean, or only documented exceptions remain. (CHK-01, CHK-02)
- [ ] Every part has `MPN`, `Manufacturer`, `LCSC` (if JLC), `Datasheet`. (BOM-01)
- [ ] Buck/LDO feedback resistors recalculated; `MPN` matches `Value`. (BOM-05)
- [ ] Capacitor voltage rating ≥ 1.5–2× working voltage (X7R ceramics lose capacitance under DC bias).
- [ ] TVS and MOSFET: Vds margin and clamp voltage checked. (MOT-07)
- [ ] Mounting holes and connector shields have nets. (LAY-07)
- [ ] BOOT, RESET and SWD are accessible. (LAY-06)

**PCB**
- [ ] Stackup matches the fab; rules within fab capability. (STK-01, DRU-01)
- [ ] DRC clean with `--schematic-parity --refill-zones`. (CHK-01)
- [ ] Power current loops minimised; enough current-carrying vias. (MOT-02, MOT-03)
- [ ] Shunt sense routed Kelvin, away from switch nodes. (MOT-05)
- [ ] Antenna keepout correct on all layers. (RF-01)
- [ ] Footprints checked against datasheets (QFN/DFN exposed pad, pin-1 orientation).
- [ ] Connector footprints match the real part. `[OBSERVED]` P1: J1 LCSC part is an XT30 but the footprint is a 5.08 mm terminal block.
- [ ] Silkscreen has name, revision, date, polarity, no typos. (SILK-01, SILK-02)
- [ ] 3D model checked for collisions and enclosure fit.

**Fabrication**
- [ ] Gerber, drill, BOM, CPL re-exported from the current revision and checked in a Gerber viewer. (FAB-01)
- [ ] Part rotation checked in the fab's assembly preview.
- [ ] Tag/release created and revision table updated. (REV-01, REV-03)

---

## Appendix A: Extracted statistics `[OBSERVED]`

| Board | Footprints | Most-used track widths (mm × segment count) | Vias (diameter/drill × count) |
|---|---|---|---|
| P1 Mini-PID-X2 | 93 | 0.2 × 343, 0.4 × 239, 0.3 × 54, 0.8 × 39, 1.5 × 12, 2.0 × 7 | 0.6/0.3 × 191 |
| P2 ESP32-EVB Rev L | 170 | 0.2032 × 1252, 1.016 × 603, 0.508 × 439 | 0.9/0.6 × 157, 0.45/0.3 × 131 |
| P3 OpenESC-20x20 | 191 | 0.2 × 637, 0.16 × 191, 0.25 × 135, 0.4 × 100 | 0.35/0.2 × 820, 0.45/0.3 × 309 |
| P4 OpenFC-Lite-Mini | 131 | 0.1 × 408, 0.2 × 216, 0.3 × 36 | 0.35/0.2 × 316 |
| P6 sc1 (4 layers) | 118 | 0.2 × 553, 0.4 × 56 | 0.7/0.3 × 141, 0.8/0.4 × 88 |
| P6 rd501 (4 layers, 2 oz) | 135 | 0.2 × 855, 0.3 × 151, 0.5 × 99 | 0.5/0.2 × 326, 0.56/0.3 × 104 |
| P6 rd321 (4 layers) | 95 | 0.2 × 564, 0.4 × 23 | 0.56/0.3 × 196 |
| P6 oi401 (2 layers) | 75 | 0.2 × 510, 0.35 × 44 | 0.56/0.3 × 79, 0.7/0.4 × 43 |
| P7 VESC 4.12 (4 layers) | 127 | 0.254 × 816, 0.635 × 278, 1.016 × 121, 1.27 × 91 | 0.8/0.5 × 113, 1.524/0.762 × 63 |

## Appendix B: Useful tools
- `kicad-cli`: ERC, DRC, Gerber/BOM export, render, STEP from the command line; CI-friendly.
- `pcbnew` Python API (KiCad's bundled Python), `kicad-skip`, `kicad_sch_api`: scripted metadata edits and design audits.
- `easyeda2kicad`: import symbol, footprint and 3D model from an LCSC number.
- Plugins: Fabrication Toolkit (JLCPCB), InteractiveHtmlBom, Round Tracks (P3 has `round-tracks-config`).
- Espressif's official library: https://github.com/espressif/kicad-libraries

## Appendix C: Rule index
| Prefix | Section | Topic |
|---|---|---|
| REPO | 2.2 | Repository layout |
| VER | 3.1 | KiCad version |
| LIB | 3.2 | Libraries |
| VAR | 3.3 | Text variables |
| NC | 3.4 | Net classes |
| STK | 4.2 | Stackup |
| DRU | 4.3 | Custom rules file |
| SCH | 5.1, 5.4 | Schematic structure |
| PWR | 5.2 | Power and ground naming |
| BOM | 5.3 | Component fields |
| LAY | 6.1 | General layout |
| MOT | 6.2 | Motor power stage |
| RF, SEN | 6.3 | RF and sensors |
| SILK | 6.4 | Silkscreen |
| CHK | 7 | Verification |
| FAB | 8 | Fabrication output |
| REV | 9.1 | Revisions |
| LIC | 9.4 | Licences |
