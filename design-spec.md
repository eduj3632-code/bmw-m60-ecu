# BMW M60 ECU — Design Specification

## 1. Project Overview

**Project:** BMW M60 V8 Engine Management ECU  
**Architecture:** Hellen-One modular KiCad platform  
**Target Board Size:** ~120 × 100 mm (accommodates all modules with I/O density)  
**ECU Target:** BMW M60 (M60V8) V8 engine — 1987–1996 production  
**Firmware:** rusEFI compatible  

---

## 2. Module Selection Rationale

### 2.1 Core Brain: `mega-mcu144` Module

**Decision:** Reuse the existing `mega-mcu144` module. **No new 'brain' module needed.**

**Rationale:**
- The M60 ECU requires substantial I/O: 8 injectors + 8 ignition coils + 2 WBO channels + 4 VR sensing channels (crank + cam per bank) + 8 analog inputs (TPS x2, MAP, IAT x2, coolant, oil, batt) + CAN bus + power management
- The `mega-mcu144` (STM32H144 variant) provides ~144 GPIO pins with abundant peripherals (4x ADC, 2x CAN, 2x UART, 2x I2C, 2x SPI, timers, PWMs)
- Compared to the `mcu` module (STM32F407VGT6, 100 pins), the H144 provides ~44 additional pins critical for the M60's I/O density
- The existing `hellen88bmw` design already successfully uses the `mega-mcu144` module for a BMW application, confirming footprints and schematic patterns are proven
- Creating a new custom brain module would duplicate existing work and violate the Hellen-One "no reinvention" principle

**STM32H144 Pin Mapping for M60:**
- INJ_1–8 → PA0–PA7 (low-side driver outputs, mapped to `output` module)
- IGN_1–8 → PB0–PB7 (coil driver outputs)
- VR_CRK_BANK1, VR_CAM_BANK1 → PI8, PI9 (timer input capture for trigger sensing)
- VR_CRK_BANK2, VR_CAM_BANK2 → PI10, PI11 (second bank trigger)
- TPS_BANK1, TPS_BANK2 → PA4, PA5 (ADC input, dual throttle bodies per bank)
- MAP → PA6 (ADC input)
- IAT_BANK1, IAT_BANK2 → PA7, PB0 (ADC input, but re-used; see layout)
- CLT → PB1 (NTC thermistor input, ADC)
- OIL → PB2 (NTC thermistor input, ADC)
- Battery Sense → PB12 (voltage divider)
- CAN_H → PB13 (CAN transceiver)
- CAN_L → PB14 (CAN transceiver)
- AUX_TX → PA9 (UART TX, TunerStudio/rusEFI)
- AUX_RX → PA10 (UART RX)
- WBO1_SPI, WBO2_SPI → dedicated SPI pins (mapped to separate WBO module)

### 2.2 Output Module: `ign8` + `output`

**Module Selection:**
- `output` module: 8x high-current low-side drivers (AUIPS2041LTR or ZXMS6005SGTA/DGTA) for fuel injectors
- `ign8` module: 8x ignition coil drivers with individual current sensing
- Both modules are proven in `hellen88bmw` and other Hellen designs

**Injector Driving:**
- 8x low-side drivers (AUIPS2041LTR preferred — 30A peak, 2.5mΩ RDS(on))
- Each injector connected to driver drain, source to GND
- Gate drive from mega-mcu144 GPIO
- Catch diodes to 5V supply (per module design)

**Ignition Coil Driving:**
- 8x coil-on-plug (M60 uses individual coils per cylinder, not waste-spark)
- Each coil: primary connected to 12V via driver transistor, ECU switches ground
- `ign8` module provides high-side driver for coil primary
- Secondary winding goes to spark plug
- Dual-fire or individual fire — M60 uses individual coil firing

### 2.3 Power12-and-5 Module

**Module:** `power12-and-5` (proven from hellen-one catalog)

**Function:**
- Battery/ignition voltage input (6–18V, protected with TVS and fuse)
- TLS115D0EJXUMA1 LDO for 5V rail (preferred per hellen-one spec)
- 12V rail for output drivers
- 3.3V LDO for digital circuitry
- Reverse polarity protection
- External input capacitors (100µF + 10µF) per power module design

**Power Architecture:**
- VIN → fuse → TVS (SMVJ15A or similar) → power12-and-5 module
- 5V → to MCU core, analog inputs, output driver logic
- 12V → to output module driver coils
- 3.3V → to MCU I/O, sensors (TPS, IAT, MAP, CLT, OIL, WBO)

### 2.4 VR Module

**Module:** `vr` (MAX9926 ASIC or discrete comparators)

**Function:**
- 4x VR channels: crank + cam per bank (2 banks = 4 channels total)
- MAX9926: differential trigger threshold, built-in hysteresis
- Discrete comparator alternative (LM339 quad comparator) if MAX9926 unavailable
- VR signals: individual toothed wheels per bank on crank and cams
- Pull-up resistors and noise filtering per VR module design
- STM32H144 captures VR signals via timer input capture pins

### 2.5 Wideband O2 Module

**Module:** `wbo` (STM32F042K6T6 + BTS3028SDR MOSFET)

**Function:**
- 2x WBO channels (one per bank)
- Each channel: separate WBO module with its own STM32F042K6T6
- BTS3028SDR MOSFET for heater control
- I2C interface to main MCU for AFR data
- Separate module per Hellen-One convention — minimal main-board pins
- LSU 4.9 or LSU 6.2 wideband lambda sensors per bank

### 2.6 CAN Bus Transceiver

**Module:** `can` (SN65HVD230 or equivalent)

**Function:**
- CAN_H / CAN_L differential pair to DB15 or M12 connector
- SN65HVD230DCW receiver with ±36V protection
- Termination resistors (120Ω) placed near connector
- Optional high-speed CAN (up to 1Mbps for TunerStudio/rusEFI communication)
- Side-channel diagnostics (error flags, overflow indicators)

### 2.7 Input Module: `dinput4` + `knock`

**Module:** `dinput4` for general digital/TPS inputs, `knock` for knock sensing

**Function:**
- 2x TPS inputs (dual throttle bodies per bank, ratiometric)
- MAP sensor input (analog)
- 2x IAT inputs (thermistor-based)
- Coolant temp sensor input
- Oil temp sensor input
- Knock sensor input (with amplification and filtering)
- All analog inputs fed to MCU ADC channels

---

## 3. Pinout Strategy

### 3.1 Main Connector (Hellen-One H144 pinout)

The M60 ECU uses a custom 144-pin or 120-pin automotive connector. The pinout is organized by function:

| Function | Pins | Description |
|----------|------|-------------|
| **Injector Drivers** | 8 | Low-side drivers (AUIPS2041LTR), INJ_1–INJ_8 |
| **Ignition Coils** | 8 | High-side drivers, IGN_1–IGN_8 |
| **VR Sensing** | 4 | Crank/cam per bank: VR_CRK_B1, VR_CAM_B1, VR_CRK_B2, VR_CAM_B2 |
| **TPS** | 2 | Dual throttle bodies, ratiometric, TPS_BANK1, TPS_BANK2 |
| **MAP** | 1 | Manifold absolute pressure |
| **IAT** | 2 | Intake air temp, Bank 1 & Bank 2 |
| **Coolant Temp** | 1 | CLT sensor, NTC |
| **Oil Temp** | 1 | OIL sensor, NTC |
| **Battery Sense** | 1 | V_BATT voltage divider |
| **CAN** | 2 | CAN_H, CAN_L (differential) |
| **WBO I2C** | 2 | SDA, SCL (shared with WBO modules) |
| **Aux/Debug** | 4 | AUX_TX, AUX_RX, DEBUG_CLK, DEBUG_SWO |
| **Power** | 4 | VIN, GND, 5V, 12V |
| **SPARE** | 4 | SPARE_1–SPARE_4 |

### 3.2 MCU GPIO Summary (STM32H144)

| Peripheral | Pins | Functions |
|------------|------|-----------|
| ADC1 | PA0–PA7, PB0–PB3 | TPS, IAT, MAP, CLT, OIL, battery sense |
| Timers (ICR) | PI8–PI11 | VR_CRK_B1, VR_CAM_B1, VR_CRK_B2, VR_CAM_B2 |
| UART | PA9–PA10 | AUX_TX, AUX_RX (TunerStudio) |
| SPI | PA5–PA7, PB3–PB5 | WBO I2C/SPI interface |
| GPIO Outputs | PA0–PA7, PB0–PB7 | INJ_1–INJ_8, IGN_1–IGN_8 |
| CAN | PB13–PB14 | CAN_H, CAN_L |
| I2C | PB6–PB7 | Secondary I2C (optional) |
| PWM | Various | Fuel pump relay, idle control, VVT |

---

## 4. Board Layer Stackup

The board uses a 4-layer stackup matching hellen-one conventions:

| Layer | Function | Thickness |
|-------|----------|-----------|
| Layer 1 (Top) | Top copper, components | 35µm |
| Layer 2 | Inner Layer 1 | 18µm |
| Layer 3 | Inner Layer 2 | 18µm |
| Layer 4 (Bottom) | Bottom copper, ground pour | 35µm |
| Dielectric | Prepreg between layers | 105µm total |

**Net Assignment:**
- **GND:** Solid pour on all layers, via fence every 200mm
- **5V:** Power plane or dedicated trace
- **12V:** Top layer trace with current limiting
- **Signal:** Layer 2 / Layer 3 (controlled impedance 50Ω for CAN, 100Ω differential)

**Via Rules:**
- GND vias: 0.3mm diameter, filled
- Signal vias: 0.3mm diameter
- Power vias: 0.5mm diameter where crossing planes

---

## 5. Module Placement Plan

### 5.1 Module Arrangement

```
+------------------------------------------------------+
|                POWER12-AND-5 (left side)             |
|  +----------------+   +----------------+   VR module   |
|  |                |   |                |   (top center)|
|  |   mega-mcu144  |---|   OUTPUT       |               |
|  |   (center)     |   |   (8 injectors)|               |
|  +----------------+   +----------------+               |
|                |             |    ^                   |
|                |             |    |                   |
|   CAN MODULE   |   WBO MODULE   |  IGN8 module        |
|   (right side) |   (top right)  |   (bottom center) |
+------------------------------------------------------+
```

### 5.2 Detailed Placement

1. **mega-mcu144** (center-left): Main MCU, sits centrally for shortest traces to all modules. LC-filtered 5V analog supply on dedicated pins.

2. **Power12-and-5** (left side): Input conditioning, LDO outputs. Proximity to connector (VIN, GND) minimizes noise. TVS and fuse near connector entry.

3. **Output module** (bottom center): 8x low-side injectors + 8x high-side ignition coils. Gate drivers near MCU, sense resistors near injector returns.

4. **ign8 module** (bottom center, overlapping with output): 8x ignition coil drivers. Separate from injector drivers to avoid cross-talk.

5. **VR module** (top center): 4x VR conditioning (MAX9926 or discrete). Near connector for VR sensor wiring. Differential pairs routed to MCU timer inputs.

6. **WBO module** (top right): 2x wideband O2 channels. Separate STM32F042K6T6 per bank. I2C bus to main MCU. Heater traces controlled for thermal stability.

7. **CAN module** (right side): SN65HVD230 transceiver, 120Ω termination, DB15 or M12 connector. Isolation capacitors near transceiver.

8. **Input module** (left side): TPS, MAP, IAT, CLT, OIL, knock sensor conditioning. Differential pairs for noisy signals.

### 5.3 I/O Pin Distribution

| Module | Pin Count | Key Signals |
|--------|-----------|-------------|
| mega-mcu144 | ~80 used | All ADC, timers, UART, CAN, SPI, PWM |
| power12-and-5 | 12 | VIN, 5V, 3.3V, GND, enable |
| output (injectors) | 16 | INJ_1–INJ_8 (low-side), driver gates |
| ign8 (coils) | 16 | IGN_1–IGN_8 (high-side), coil sense |
| vr | 8 | VR_CRK_B1, VR_CAM_B1, VR_CRK_B2, VR_CAM_B2 |
| wbo | 8 (per channel) | I2C, heater control, lambda data |
| can | 6 | CAN_H, CAN_L, termination, diagnostics |
| dinput4/knock | 10 | TPS x2, MAP, IAT x2, CLT, OIL, knock |

**Total utilized:** ~144 pins (full H144 utilization)  
**Board utilization:** ~90% of available I/O (required for complete M60 coverage)

---

## 6. Open Decisions & Design Gaps

### 6.1 VR Sensor Type: MAX9926 vs Discrete

**Decision pending:** Use MAX9926 ASIC or discrete comparators (LM339)?

- **MAX9926:** Single-chip solution, built-in hysteresis, differential inputs, simpler PCB design. Requires SPI configuration.
- **Discrete:** LM339 quad comparator, more external components (resistors, capacitors for hysteresis), proven in many hellen designs.
- **Recommendation:** Start with MAX9926 for simplicity; add discrete fallback in revision B if supply chain issues arise.

### 6.2 Wideband O2 Sensor Model

**Decision pending:** LSU 4.9 vs LSU 6.2 vs universal?

- LSU 4.9: Older, widely available, compatible with most ECU platforms.
- LSU 6.2: Newer, faster response, requires heated oxygen sensor simulator for bench testing.
- **Recommendation:** Specify LSU 4.9 for initial release; LSU 6.2 as revision option.

### 6.3 Ignition Coil Type

**Decision confirmed:** Coil-on-plug (individual coils per cylinder), NOT waste-spark.

- M60 uses individual coil-per-cylinder layout
- Waste-spark would require only 4 coils but stress all 8 cylinders with dual-fire
- Individual fire allows independent timing, better combustion efficiency, and reduced misfire detection latency
- **Rationale:** M60 factory architecture supports coil-on-plug; waste-spark would require major firmware changes and reduce reliability.

### 6.4 Connector Type

**Decision pending:** 120-pin vs 144-pin automotive connector?

- 120-pin: Compact, existing tooling, but fewer spare pins
- 144-pin: Full pin count, matches mega-mcu144 footprint, but higher cost
- **Recommendation:** 144-pin for initial release (future-proof), with optional 120-pin revision if cost becomes a constraint.

### 6.5 Board Revisions

- **Revision A:** Initial release, all modules validated, basic firmware support (inj, ign, can, wbo)
- **Revision B:** ADD knock control, ADD electronic throttle (if required), improved VR filtering, refined PCB stackup
- **Revision C:** ADD CAN-FD support, ADD secondary O2 sensor, revised power stage components

### 6.6 Firmware Decisions (out of scope for hardware, documented)

- rusEFI core with M60-specific defines
- VR trigger configuration: 36-1 or 60-2 pattern per bank (M60 uses individual toothed wheels)
- Ignition timing: Individual coil fire, dwell time management
- Fuel injection: Simultaneous or batch mode (sequential preferred)
- WBO integration: lambda target control, sensor warmup sequences
- CAN: TunerStudio/rusEFI .ini file generation, parameter server

---

## 7. File Summary

| File | Description |
|------|-------------|
| `/opt/data/projects/bmw-m60-ecu/bmw-m60-ecu.kicad_pro` | Project file with net classes, text variables |
| `/opt/data/projects/bmw-m60-ecu/bmw-m60-ecu.kicad_sch` | Schematic title block, pin definitions, module references |
| `/opt/data/projects/bmw-m60-ecu/bmw-m60-ecu.kicad_pcb` | Blank PCB with layer stackup, net classes, title block |
| `/opt/data/projects/bmw-m60-ecu/design-spec.md` | This design specification |

---

## 8. Module Dependency Map

```
bmw-m60-ecu
├── mega-mcu144 (brain: STM32H144, ADC, timers, UART, CAN, SPI, PWM)
│   ├── power12-and-5 (5V/3.3V regulation, LDO)
│   ├── output (8x low-side injector drivers, AUIPS2041LTR)
│   ├── ign8 (8x ignition coil drivers)
│   ├── vr (4x VR conditioning, MAX9926/discrete)
│   ├── wbo (2x wideband O2, STM32F042K6T6 + BTS3028SDR)
│   ├── can (SN65HVD230 transceiver, DB15/M12 connector)
│   └── dinput4/knock (TPS, MAP, IAT, CLT, OIL, knock sensor)
└── connectors (custom 144-pin automotive, pinout defined in main.yaml)
```

---

## 9. Checklist for CI/GPH Actions

- [ ] Fork hellen-example, create frame PCB
- [ ] Place all Hellen modules per placement plan
- [ ] Run DRC after module placement
- [ ] Verify net classes assigned per hellen-one conventions
- [ ] Export gerbers via `kicad-cli pcb export gerbers --check-zones`
- [ ] Generate BOM with `hellen-one-kicad-bom-plugin.py`
- [ ] Validate CPL (pick-and-place) export
- [ ] Run test frame verification in `tests/` directory
- [ ] Commit/push triggers CI Gerber export pipeline
