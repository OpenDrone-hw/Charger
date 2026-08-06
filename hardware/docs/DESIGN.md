# Charger Design Notes

Detailed design description of both boards. Values are extracted from the KiCad design files: `hardware/Charger.kicad_sch` (+ `main.kicad_sch`, `cell.kicad_sch`), `hardware/Charger.kicad_pcb`, `hardware-manager/Manager.kicad_sch` (+ `main.kicad_sch`, `cell.kicad_sch`). Verified defects, the layout state and the fabrication-readiness verdict are in [SCHEMATIC_REVIEW.md](SCHEMATIC_REVIEW.md); several subcircuits below are incompletely wired as committed.

## System concept

A distributed charging system: one Manager hub and multiple headless Charger nodes, each charging one pack. Both boards share the same charge, sense and balance core: a BQ25758 buck-boost charge controller, a TLA2528 8-channel I2C ADC reading cumulative resistor-divider taps for per-cell voltage, and a TCA9554A I2C expander driving per-cell bleed-balance FETs. The ESP32 radios are intended for ESP-NOW coordination between Manager and nodes; no firmware exists in this repo. Design targets: automatic cell-count detection, charge current chosen from measured pack internal resistance, and Li-ion / LiPo / LiHV / LiFe chemistry support.

## Charger node (`hardware/`)

Headless 2-6S charging node, USB-C powered, charge only, 3 A design target.

| Block | Part | Notes |
|---|---|---|
| MCU | ESP32-C3-MINI-1-N4 (U8) | Wi-Fi/BLE module, UART0 programming header |
| USB-PD sink | HUSB238A (U1) | PD 3.0 sink controller, I2C |
| Charge controller | BQ25758 (U2) | Buck-boost, I2C, 5 mOhm output sense |
| Logic rail | TPS560430 (U7) | Buck, +3V3 (600 mA) |
| Cell ADC | TLA2528 (U3) | 8-channel, I2C, cumulative tap dividers |
| Balance expander | TCA9554A (U4) | Drives 6 bleed channels |
| OV protection | LMV331 (U6) + TL432 + AP40P04G (Q18) | Comparator-driven input cutoff P-FET |
| Input clamps | SMBJ30A, SMBJ28A TVS | Rail and battery side |

Cell sheet (`cell.kicad_sch`, 6 instances): per-cell bleed balancer, SI2319 P-FET switching a 27 Ohm 2512 bleed resistor across the cell, gate level-shifted through an MMBT3904 with a 12 V zener Vgs clamp. Cell voltages are read as cumulative divider taps into the TLA2528; per-cell voltage is the difference of adjacent taps, so the 0.1% thin-film divider resistors are mandatory.

Connectors: USB-C (TYPE-C-31-M-12, PD input), XT60 male and female (DC input and pack), 7-pin JST XH balance connector, 6-pin 2.54 mm programming header.

## Manager (`hardware-manager/`)

2-8S hub with UI, 10 A charge path design target, bidirectional (charge plus USB-PD discharge) as design target. Schematic only, no PCB yet.

| Block | Part | Notes |
|---|---|---|
| MCU | ESP32-S3-WROOM-1-N8R2 (U15) | Drives display and encoder UI |
| USB-PD controller | TPS26750 (U1) | PD 3.1, config EEPROM on dedicated I2C bus |
| EPR front end | TPD4S480 (U16) | 48 V EPR VBUS protection and voltage translation |
| Power-path driver | SiLM2660 (U17) + 2x SFS08R03GNF | Back-to-back 80 V / 3.3 mOhm FETs, 240 W path |
| Input OR-ing | 3x LM74700 (U3, U4, U5) | USB-C, XT60 and solar inputs, ideal-diode controllers |
| Pack reverse protection | LM74700 (U9) + SFS08R03GNF (Q27) | Ideal-diode controller and FET at the pack XT60 |
| Charge controller | BQ25758 (U6) | Buck-boost, 5 mOhm sense, IOUT programmed to 10 A, fSW ~450 kHz |
| Power stage | 4x SFS08R03GNF (Q7-Q10) | Buck and boost legs |
| 12 V rail | TPS54160 (U12) | Gate-drive supply (DRV_SUP) |
| 5 V / 3V3 rails | 2x TPS54202 (U13, U14) | Logic and USB source rails |
| Cell ADC / balance | TLA2528 (U7) + TCA9554A (U8) | 8 bleed channels |
| UI | FPC display connector (J4) + EC11 encoder + tactile switches | SPI LCD |

Cell sheet (`cell.kicad_sch`, 8 instances): same SI2319 bleed topology as the node.

Connectors: USB-C (TYPE-C-31-M-12, EPR), XT60 male and female, 9-pin JST XH balance connector, FPC display connector.

## Libraries

Placed parts come from three sources:

- Project-local `sourced` library (`libs/sourced.kicad_sym`, `libs/sourced.pretty/`, `libs/sourced.3dshapes/`), listed in each project's `sym-lib-table` and `fp-lib-table`. Holds the vendor-imported parts: ICs, connectors, FETs, inductors, module footprints.
- KiCad stock libraries (`Device:`, `power:`, `Resistor_SMD:`, `Capacitor_SMD:`, `Diode_SMD:`, `Package_TO_SOT_SMD:`, `Connector_Generic:`). These carry all passives and generic symbols and are the majority of placed parts on both boards. They are global KiCad libraries, not listed in either project's lib tables, so a clone opened without KiCad's stock libraries will not resolve most of the design. The OpenDrone convention is project-local libraries only; making this repo self-contained is a migration task, not done.
- Shared `Incutec` library from the `libs/KiCad-Library` submodule. Referenced by both projects' lib tables, but no placed part uses it yet.

`hardware/libs` and `hardware-manager/libs` are byte-identical copies of the same `sourced` library, 116 MB each. Collapsing them to one copy at repo root also needs the `${KIPRJMOD}/libs/sourced.3dshapes/...` 3D model paths embedded in `hardware/Charger.kicad_pcb` and in the `.kicad_mod` files rewritten, so it is a KiCad-side edit, not a file move. Separately, 14 of the 66 footprints in each `sourced.pretty` point their 3D models at an absolute path under an old checkout location (`/Users/stan/OpenDrone/Charger/hardware/libs/...`), which no longer resolves; they need repointing at `${KIPRJMOD}`.

## Firmware

No firmware exists in this repo. The intended split is hardware under CERN-OHL-S-2.0 and firmware under MIT once published.
