# Agent notes

Facts for AI agents working in this repo.

- Two KiCad projects (Charger node files in KiCad 10 format, Manager in KiCad 9). Charger node: root `hardware/Charger.kicad_sch`, main sheet `hardware/main.kicad_sch`, cell sheet `hardware/cell.kicad_sch` (6 instances), board `hardware/Charger.kicad_pcb` (layout in progress). Manager: root `hardware-manager/Manager.kicad_sch`, main sheet `hardware-manager/main.kicad_sch`, cell sheet `hardware-manager/cell.kicad_sch` (8 instances), no PCB.
- Clone with `git clone --recursive`; the `libs/KiCad-Library` submodule is referenced by both project lib tables for shared parts. Each project also has a project-local `sourced` library under its `libs/`.
- Never edit `.kicad_*` files as text. Use kicad-skip or the pcbnew API, and only for metadata (text variables, symbol BOM/doc fields). Never change nets, placement, or component values.
- Checks:

```
kicad-cli sch erc --exit-code-violations hardware/Charger.kicad_sch
kicad-cli sch erc --exit-code-violations hardware-manager/Manager.kicad_sch
kicad-cli pcb drc --exit-code-violations hardware/Charger.kicad_pcb
```

- Known schematic defects, including a broken cell-sheet annotation on the Charger node root, are catalogued in `hardware/docs/SCHEMATIC_REVIEW.md`. Neither board is fab-ready.
- Fabrication exports land in `hardware/production/` (gitignored), generated with the KiCad Fabrication Toolkit; none exist yet.
- Docs are deterministic: current fact only, no TODOs or plans.
- `main` is protected; push feature branches and open PRs.
