# BMW M60 ECU — Hellen-One Board

Engine management board for BMW M60 (M60V8) V8 engine using the [Hellen-One](https://github.com/andreika-git/hellen-one) modular platform.

## Overview

- **MCU:** STM32H144 (mega-mcu144 module)
- **Injectors:** 8× low-side drivers (AUIPS2041LTR)
- **Ignition:** 8× coil-on-plug drivers
- **Wideband O2:** 2× channels (one per bank)
- **VR Sensing:** 4× channels (crank + cam per bank)
- **Analog Inputs:** TPS ×2, MAP, IAT ×2, CLT, OIL, batt voltage
- **Communication:** CAN bus (SN65HVD230) + UART for TunerStudio/rusEFI
- **Board Size:** ~120 × 100 mm
- **Layers:** 4-layer PCB

## Modules Used

| Module | Function |
|--------|----------|
| `mega-mcu144` | STM32H144 brain, ADC, timers, UART, CAN, SPI |
| `power12-and-5` | 12V→5V/3.3V regulation (TLS115D0EJXUMA1 LDO) |
| `output` | 8× injector low-side drivers |
| `ign8` | 8× ignition coil drivers |
| `vr` | 4× VR conditioning (MAX9926 / discrete) |
| `wbo` | 2× wideband O2 (STM32F042K6T6 + BTS3028SDR) |
| `can` | CAN transceiver + connector |
| `dinput4` + `knock` | Analog input conditioning |

See [design-spec.md](./design-spec.md) for full architecture details.

## Board Creation

This board follows the Hellen-One board creation workflow:

1. Fork of [`rusefi/hellen-example`](https://github.com/rusefi/hellen-example)
2. Frame PCB with modules placed
3. Push triggers CI → Gerber export + BOM generation
4. Download fabrication files from `boards/` folder
5. Order from JLCPCB or equivalent

## CI / Automation

Pushing to `main` triggers the hellen-one [create-board](https://github.com/andreika-git/hellen-one/actions) workflow which:
- Exports KiCad gerbers via `kicad-cli`
- Merges module gerbers into the frame layout
- Generates BOM and CPL
- Places outputs in the `boards/` folder

## Open Design Decisions

See [design-spec.md §6](./design-spec.md#6-open-decisions--design-gaps) for:
- VR sensor type (MAX9926 vs discrete)
- WBO sensor model (LSU 4.9 vs 6.2)
- Connector pin count (120 vs 144)
- Board revision plan (A/B/C)

## Related

- [Hellen-One Wiki](https://github.com/andreika-git/hellen-one/wiki)
- [rusEFI Wiki](https://github.com/rusefi/rusefi/wiki)
- [hellen88bmw](https://github.com/rusefi/hellen88bmw) — similar BMW board
