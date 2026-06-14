# System architecture — distributed USB-C charging nodes

**Concept.** Not one big charger. An **array of cheap "charging nodes"**, each fed
by an **off-the-shelf USB-C PD brick** (we don't design the PD source). Each node:
takes USB-C PD in → outputs **2–6S on JST-XH** → **auto-picks charge current from
measured internal resistance** → **balances cells** → reports status **wirelessly**.
Optionally one **"manager" node has a screen** and shows nearby nodes' status.

Target: FPV packs. **3A stack current** (≈65W at 6S, less at lower S — see
`research/cell-balancing.md`) is plenty; IR-based current auto-derates small packs.

## Per-node block diagram

```
[off-shelf USB-C PD brick] --USB-C--> [PD sink + buck-boost charger: IP2366]
                                              |  charge (<=3A)
                                              v
                                       [JST-XH out: charge + balance]
                                              ^  per-cell sense + bleed
                                       [2-6S balancing AFE]
   [ESP32-C3/C6] --I2C--> IP2366 + AFE  (runs IR->current algo, wireless, status)
```

## Building blocks

| Block | Part | State |
|---|---|---|
| PD sink + buck-boost power stage | **IP2366** (PD3.1, 2–6S, I²C, ≤3A config) | chosen |
| Cell sense + passive balancing + protection | 2–6S **AFE** (per-cell V for IR + balance + OV/UV/OT) | **research** |
| Brains + wireless | **ESP32-C3/C6** (Wi-Fi + BLE + ESP-NOW) | chosen (family) |
| Manager variant | node + small display (OLED/LCD) | optional SKU |
| Connectors | USB-C (in), **JST-XH** (charge+balance, ≤3A) | set |

IP2366 is the efficient power stage; the AFE gives per-cell data + balancing
(IP2366 only sees the whole stack); the ESP is the supervisor + radio.

## Charge current from internal resistance

ESP applies a current step, reads ΔV/ΔI **per cell** (AFE ADC) and **pack** (IP2366
ADC) → estimates IR → picks a charge current bounded by: C-rate (from IR/health),
thermal limit, and the **3A connector ceiling** → writes IP2366's current setpoint
over I²C. Result: big healthy 6S packs charge near 3A; tiny/high-IR whoop packs are
automatically throttled. Cell count + chemistry auto-detected (cell voltages) or set.

## Wireless / fleet

Each node broadcasts status (cell count, per-cell V, SoC, current, temp, IR, ETA)
over **ESP-NOW** (no router needed) and/or BLE. Consumers:
- **Manager node** (with screen) — pops up nearby charging nodes + their progress.
- **Phone app** (BLE/Wi-Fi) — same, optional.

## Safety

Per-cell OV/UV/OT + pack OC/SC from the AFE; USB-C input OVP; output fuse; firmware
limits enforce the 3A ceiling and IR-derived max current. Never charge above XH 3A.

## Open / to-research

1. **AFE that flexes 2–6S** with per-cell ADC + passive balancing, JLCPCB-sourceable
   (BQ769x0 family is typically ≥3S; BQ76952 is 3–16S → 2S is the gap; find a 2–6S
   path or accept 3–6S). Run `component-research` → `research/cell-balancer.md`.
2. Confirm IP2366 I²C: current-setpoint write + ADC readback resolution (for IR).
3. ESP32-C3 vs C6 (C6 = Wi-Fi6/Thread; C3 cheaper). Pick per radio needs.
4. Manager: dedicated SKU vs every node display-capable (cost vs flexibility).
5. Cost target per node (aim simple/cheap: IP2366 + AFE + ESP32-C3 + passives).

## Milestones

- **M0** research AFE; confirm IP2366 I²C control/readback.
- **M1** single-node schematic (IP2366 + AFE + ESP32-C3) → ERC clean.
- **M2** firmware: IR→current algorithm + balancing supervision.
- **M3** wireless (ESP-NOW) + manager-node screen/app.
- **M4** layout + JLCPCB fab; validate on real FPV packs.
