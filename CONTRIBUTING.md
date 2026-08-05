# Contributing

## Setup

```
git clone --recursive https://github.com/incutec-hw/Charger.git
```

The `--recursive` flag pulls the `libs/KiCad-Library` submodule, which both project lib tables reference for shared parts. Use KiCad 10.

## Workflow

- `main` is protected. Work on a feature branch and open a pull request.
- Edit schematics and the PCB in KiCad only. Never text-edit `.kicad_sch`, `.kicad_pcb`, or other `.kicad_*` files.
- New shared parts (symbols, footprints, 3D models) go to [incutec-hw/KiCad-Library](https://github.com/incutec-hw/KiCad-Library) via PR there, not into the project-local `sourced` libraries.
- ERC and DRC must be clean before opening a PR:

```
kicad-cli sch erc --exit-code-violations hardware/Charger.kicad_sch
kicad-cli sch erc --exit-code-violations hardware-manager/Manager.kicad_sch
kicad-cli pcb drc --exit-code-violations hardware/Charger.kicad_pcb
```

Both schematics currently carry known defects, catalogued in [hardware/docs/SCHEMATIC_REVIEW.md](hardware/docs/SCHEMATIC_REVIEW.md). Fixes for listed findings are welcome.

## Documentation

Docs state current fact only: no TODOs, no plans, no aspirational content.

## Licensing

Contributions are licensed under CERN-OHL-S-2.0, the same license as the project.

## Questions

Open a GitHub issue.
