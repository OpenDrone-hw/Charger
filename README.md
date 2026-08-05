# Charger

Open-source distributed USB-C charging system for FPV packs (incutec OpenDrone line).
Two boards sharing one Charge + Sense + Balance core:

- **Slave** (`hardware/`) — cheap headless charging node, 2–6S, ≤3 A, charge-only. ESP32-C3.
- **Manager** (`hardware-manager/`) — hub with screen + UI, 2–8S, ~10 A, bidirectional
  (charge + USB-PD discharge), USB-C 240 W EPR + XT60 + solar. ESP32-S3. Coordinates
  slaves over ESP-NOW.

Both auto-detect cell count, pick charge current from each pack's internal resistance,
balance the cells, and support Li-ion / LiPo / LiHV / LiFe.

## Repository
- `hardware/` — slave KiCad project: `Charger.kicad_sch` → `main.kicad_sch` + 6× `cell.kicad_sch`.
- `hardware-manager/` — manager KiCad project: `Manager.kicad_sch` → `main.kicad_sch` + 8× `cell.kicad_sch`.
- `libs/` (per project) — project-local symbols, footprints, and 3D models.
- `datasheets/` — component datasheets by MPN + LCSC.

The KiCad schematics are the single source of truth for the circuit.

## License
Hardware: CERN-OHL-S. Firmware: MIT. (Not USB-IF certified — DIY / open-source use.)
