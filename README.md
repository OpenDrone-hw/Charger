# Charger

Open-source distributed USB-C charging system for FPV battery packs, part of the incutec OpenDrone line. Two boards share one charge, sense and balance core built around the TI BQ25758 buck-boost charge controller: the **Charger node** (`hardware/`), a headless 2-6S charging node, and the **Manager** (`hardware-manager/`), a 2-8S hub with display and encoder UI. Designed in KiCad (Charger node in KiCad 10 format, Manager in KiCad 9) for JLCPCB assembly.

Specifications, block-by-block part choices, power tree and cell sense/balance detail: [hardware/docs/DESIGN.md](hardware/docs/DESIGN.md).

## Status

**Prototype pending.** Neither board has been fabricated. The Charger node has a schematic and a PCB with footprints placed but nothing routed; the Manager is schematic only. Both boards currently fail the schematic review and are not ready for fabrication. Defect list, layout state and readiness verdict: [hardware/docs/SCHEMATIC_REVIEW.md](hardware/docs/SCHEMATIC_REVIEW.md).

## Repository layout

| Path | Contents |
|---|---|
| `hardware/` | Charger node KiCad 10 project: schematics, PCB, `libs/` ([libraries](hardware/docs/DESIGN.md#libraries)) |
| `hardware/docs/` | Design documentation ([DESIGN.md](hardware/docs/DESIGN.md), [SCHEMATIC_REVIEW.md](hardware/docs/SCHEMATIC_REVIEW.md)) |
| `hardware-manager/` | Manager KiCad 9 project: schematics, `libs/` (no PCB yet) |
| `libs/KiCad-Library` | Shared Incutec symbol/footprint/3D library (git submodule) |

`datasheets/` directories are gitignored; datasheets are fetched on demand and do not ship with the repo.

## Design entry points

- Charger node root schematic: `hardware/Charger.kicad_sch` (main sheet `hardware/main.kicad_sch` plus 6 instances of `hardware/cell.kicad_sch`)
- Charger node board: `hardware/Charger.kicad_pcb` (footprints placed, no outline, unrouted)
- Manager root schematic: `hardware-manager/Manager.kicad_sch` (main sheet `hardware-manager/main.kicad_sch` plus 8 instances of `hardware-manager/cell.kicad_sch`)

Which symbol and footprint libraries the projects use: [DESIGN.md, Libraries](hardware/docs/DESIGN.md#libraries).

## Build and export

```
git clone --recursive https://github.com/OpenDrone-hw/Charger.git
```

Open `hardware/Charger.kicad_pro` or `hardware-manager/Manager.kicad_pro` in KiCad 10. Production exports (gerbers, BOM, CPL) are generated with the [KiCad Fabrication Toolkit](https://github.com/bennymeg/Fabrication-Toolkit) plugin; none exist yet. Headless checks use `kicad-cli`:

```
kicad-cli sch erc --exit-code-violations hardware/Charger.kicad_sch
kicad-cli sch erc --exit-code-violations hardware-manager/Manager.kicad_sch
kicad-cli pcb drc --exit-code-violations hardware/Charger.kicad_pcb
```

The `pcb drc` run is not a meaningful check yet: the Charger node board has no outline and no routing, so it passes without proving anything.

## Manufacturing

Target fab is JLCPCB assembly with LCSC parts, exported with the Fabrication Toolkit into `hardware/production/` (gitignored). No production exports have been generated and nothing has been fabricated.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

Hardware licensed under [CERN-OHL-S-2.0](https://ohwr.org/cern_ohl_s_v2.txt). See [LICENSE](LICENSE). Not USB-IF certified; intended for DIY and open-source use.
