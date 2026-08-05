# Charger

Open-source distributed USB-C charging system for FPV battery packs, part of the incutec OpenDrone line. Two boards share one charge, sense and balance core built around the TI BQ25758 buck-boost charge controller: the **Charger node** (`hardware/`), a headless 2-6S charging node with an ESP32-C3 and USB-C PD input, and the **Manager** (`hardware-manager/`), a 2-8S hub with display and encoder UI, ESP32-S3, and USB-C PD EPR (240 W), XT60 and solar inputs. Designed in KiCad (Charger node in KiCad 10 format, Manager in KiCad 9) for JLCPCB assembly. Full design detail: [hardware/docs/DESIGN.md](hardware/docs/DESIGN.md).

## Status

**Prototype pending**, schematic and first PCB layout in progress, 2026-08-05.
The Charger node has a schematic and a PCB layout in progress; the Manager is schematic only. Neither board has been fabricated. Known schematic defects are tracked in [hardware/docs/SCHEMATIC_REVIEW.md](hardware/docs/SCHEMATIC_REVIEW.md); both boards currently fail that review and are not ready for fabrication.

## Specifications

| Parameter | Charger node | Manager |
|---|---|---|
| Role | Headless charging node | Hub with display and encoder UI |
| MCU | ESP32-C3-MINI-1-N4 | ESP32-S3-WROOM-1-N8R2 |
| Charge controller | BQ25758 buck-boost | BQ25758 buck-boost |
| Pack | 2-6S, per-cell sense and bleed balancing | 2-8S, per-cell sense and bleed balancing |
| Charge current | 3 A (design target) | 10 A (design target) |
| Power input | USB-C PD sink (HUSB238A), XT60 DC | USB-C PD EPR 240 W (TPS26750 + TPD4S480), XT60 and solar, OR'd via LM74700 ideal diodes |
| Direction | Charge only | Charge and USB-PD discharge (design target) |
| Chemistries | Li-ion, LiPo, LiHV, LiFe (design target) | Li-ion, LiPo, LiHV, LiFe (design target) |

Part-level detail (charge core, PD front ends, power tree, cell sense and balance) is in [hardware/docs/DESIGN.md](hardware/docs/DESIGN.md).

## Repository layout

| Path | Contents |
|---|---|
| `hardware/` | Charger node KiCad 10 project: schematics, PCB, project-local `sourced` library |
| `hardware/docs/` | Design documentation ([DESIGN.md](hardware/docs/DESIGN.md), [SCHEMATIC_REVIEW.md](hardware/docs/SCHEMATIC_REVIEW.md)) |
| `hardware-manager/` | Manager KiCad 9 project: schematics, project-local `sourced` library (no PCB yet) |
| `libs/KiCad-Library` | Shared Incutec symbol/footprint/3D library (git submodule) |

`datasheets/` directories are gitignored; datasheets are fetched on demand and do not ship with the repo.

## Design entry points

- Charger node root schematic: `hardware/Charger.kicad_sch` (main sheet `hardware/main.kicad_sch` plus 6 instances of `hardware/cell.kicad_sch`)
- Charger node board: `hardware/Charger.kicad_pcb` (layout in progress)
- Manager root schematic: `hardware-manager/Manager.kicad_sch` (main sheet `hardware-manager/main.kicad_sch` plus 8 instances of `hardware-manager/cell.kicad_sch`)

Each project's lib tables reference its project-local `sourced` library (`libs/sourced.kicad_sym`, `libs/sourced.pretty/`, `libs/sourced.3dshapes/`) and the shared `Incutec` library from the `libs/KiCad-Library` submodule, used for new parts.

## Build and export

```
git clone --recursive https://github.com/incutec-hw/Charger.git
```

Open `hardware/Charger.kicad_pro` or `hardware-manager/Manager.kicad_pro` in KiCad 10. Production exports (gerbers, BOM, CPL) are generated with the [KiCad Fabrication Toolkit](https://github.com/bennymeg/Fabrication-Toolkit) plugin; none exist yet. Headless checks use `kicad-cli`:

```
kicad-cli sch erc --exit-code-violations hardware/Charger.kicad_sch
kicad-cli sch erc --exit-code-violations hardware-manager/Manager.kicad_sch
kicad-cli pcb drc --exit-code-violations hardware/Charger.kicad_pcb
```

## Manufacturing

Target fab is JLCPCB assembly with LCSC parts, exported with the Fabrication Toolkit into `hardware/production/` (gitignored). No production exports have been generated and nothing has been fabricated. Revision history: [CHANGELOG.md](CHANGELOG.md).

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

Hardware licensed under [CERN-OHL-S-2.0](https://ohwr.org/cern_ohl_s_v2.txt). See [LICENSE](LICENSE). Not USB-IF certified; intended for DIY and open-source use.
