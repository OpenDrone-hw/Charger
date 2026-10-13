# Charger

Distributed USB-C charging system for FPV packs: a headless charger node per
pack and a manager hub, sharing one charge, sense and balance core. No
schematic or layout exists: `hardware/` holds only the pinned `KiCad-Library`
submodule. Product intent, targets, constraints, prior art and the open design
questions are in [README.md](README.md). They are targets, not
specifications; do not restate them here and do not describe either board as
designed, tested or released.

## Repo

| | |
|---|---|
| Status | See the `status-*` topic on the repo. Never written here. |
| Designed in | KiCad 10 |
| KiCad project | None. Each board starts from [hardware-template](https://github.com/OpenDrone-hw/hardware-template) and lands in `hardware/`. |
| Earlier pass | Tag `pre-reset-2026-08-13`: reviewed as not functional, reference for the thinking only. |
| Shared library | `hardware/KiCad-Library/`, submodule of [OpenDrone-hw/KiCad-Library](https://github.com/OpenDrone-hw/KiCad-Library), nickname `OpenDrone`; 3D models and exact component datasheets resolve through the project text variable `OPENDRONE_LIB` |
| License | CERN-OHL-S-2.0 |

## Environment

There is no schematic or board to check yet. Once a project exists, the ERC,
DRC and netlist commands are the template's, on the `.kicad_sch` and
`.kicad_pcb` in `hardware/`. On macOS `kicad-cli` is at
`/Applications/KiCad/KiCad.app/Contents/MacOS/kicad-cli`; `KPY` is KiCad's
bundled Python,
`/Applications/KiCad/KiCad.app/Contents/Frameworks/Python.framework/Versions/Current/bin/python3`.

## Rules

Identical in every OpenDrone board repo. Do not edit here; edit the template.

- **Never text-edit** `.kicad_sch`, `.kicad_pcb` or `.kicad_dru`. Use KiCad, or
  kicad-skip / the pcbnew API for scripted changes. `.kicad_pro` is JSON and may
  be edited directly for metadata.
- **Metadata yes, connections no.** An agent may write BOM and documentation
  fields (MPN, Manufacturer, LCSC, Cost, Datasheet, text variables). An agent
  may not change nets, wiring, routing, placement, footprint assignment, or any
  value that changes the circuit.
- **Close KiCad before any write to a KiCad file.** KiCad caches library tables
  at process start and overwrites files on save.
- **Reuse before you draw.** Check the `OpenDrone` library and its
  `PARTS-USED.md` first. If the part is there we have already sourced,
  footprinted and shipped it, and its symbol links to the exact committed
  datasheet: place it from `OpenDrone`. Draw a new part into `lib` only when
  the catalogue has nothing that fits, imported with
  `easyeda2kicad` from its LCSC number. Pulling a newer catalogue is a
  deliberate, reviewed commit: `git submodule update --remote
  hardware/KiCad-Library`, then DRC.
- **One person holds a board layout at a time.** KiCad files do not merge. Say
  on Discord that you are taking it. See [CONTRIBUTING.md](CONTRIBUTING.md).
- **Run ERC and DRC before every pull request.** Existing approved findings
  may remain; a new type or increased count must be reviewed before merge.
  Commands are in Environment above.

## By task

Board-specific paths are in Environment above. `KPY` is KiCad's bundled
Python named there.

- Start the design: only on request, and after the README "Design questions" on bus, power budget and safety are answered. Create the KiCad project from hardware-template in `hardware/`, then replace the Environment section above with the template's commands.
- Add a part: place it from the `OpenDrone` library if `hardware/KiCad-Library/PARTS-USED.md` lists it; otherwise import it into `lib` with `$KPY <hardware-tooling>/hardware/kicad/import_part.py` (read `--help` first), KiCad closed.
- Update the shared library: `git submodule update --remote hardware/KiCad-Library`, commit as its own reviewed change; run DRC once a board exists.
- Look at the earlier schematic pass: `git show pre-reset-2026-08-13 --stat`; read it as history, never continue from it.
