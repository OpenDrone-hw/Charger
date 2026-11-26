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
| KiCad project | None. Each board starts from [hardware-template](https://github.com/OpenDrone-hw/hardware-template) and lands in `hardware/`. |
| Earlier pass | Tag `pre-reset-2026-08-13`: reviewed as not functional, reference for the thinking only. |

## Rules

The shared [Board rules](https://github.com/OpenDrone-hw/hardware-template/blob/main/AGENTS.md#board-rules) apply. Only what differs for this board is listed here.

## By task

- Start the design: only on request, and after the README "Design questions" on bus, power budget and safety are answered. Create the KiCad project from hardware-template in `hardware/`.
- Look at the earlier schematic pass: `git show pre-reset-2026-08-13 --stat`; read it as history, never continue from it.
