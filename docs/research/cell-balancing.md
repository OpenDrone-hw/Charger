# Cell balancing — open rework

**2026-06-14.** Gap found after selecting IP2366.

## The gap

IP2366 (and IP2368) charge the **whole 2–6S series stack as one** — they regulate
pack voltage/current but do **not** balance individual cells. A multi-cell Li-ion
pack needs per-cell balancing: without it, cell mismatch grows each cycle and a
weak cell can be overcharged/over-discharged → capacity loss and a safety risk.
So the charger as drawn around IP2366 alone is **incomplete for bare packs**.

## Two ways to resolve it (decision needed)

1. **Pack carries its own BMS (charger stays simple).** Spec the product for
   smart/protected packs that balance internally. IP2366 alone is then fine; the
   charger just delivers CC/CV to the pack. Cheapest/simplest, but constrains what
   batteries the user can attach.
2. **Charger includes balancing + protection (AFE/BMS front-end).** Add a
   multi-cell analog front-end with integrated balancing alongside IP2366:
   - per-cell sense taps (2–6 cell wires) + balancing (passive bleed FET/resistor
     per cell, or active),
   - candidate AFEs covering up to 6S: TI **BQ76920** (3–5S) / **BQ76930** (6–10S),
     standalone **BQ77915** (≤5S, no MCU), or ADI **LTC6811**-class. Most need an
     MCU/I²C; some standalone parts balance + protect without firmware.
   Supports bare packs, but adds an IC, per-cell wiring, balancing components, and
   possibly an MCU.

## Rework implied (path 2)

- Add the balancer/AFE + per-cell sense + balancing network to the schematic.
- Add a 2–6 pin balance connector (JST-XH style) next to the main pack terminal.
- Re-check board area / cost against the "simple, elegant, cheap" goal.
- IP2366 still does PD + buck-boost charge/discharge; the AFE adds balancing +
  per-cell protection on top.

## Next

Decide path 1 vs 2 (depends on target batteries). If path 2, run the
`component-research` skill on "2–6S Li-ion AFE with integrated cell balancing,
JLCPCB-sourceable, minimal external parts" and produce
`docs/research/cell-balancer.md`. Cross-check that the chosen full-charge voltage
and cell count match IP2366's resistor config.
