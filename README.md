# Charger — USB-PD LiPo charger (2–6S, 140W)

A simple, elegant, cheap USB-PD charger.

- **Input:** USB-C, USB-PD 3.1 EPR (up to 28V/5A = 140W).
- **Battery:** 2–6S Li-ion / LiFePO₄ (8.4–25.2V). *(1S is out of scope — the
  controller is 2S-minimum, and 1S @ 140W isn't sensible.)*
- **Power:** 140W max.
- **Topology:** single-inductor synchronous buck-boost (PD voltage straddles the
  battery voltage, so buck-boost is mandatory).

## Selected controller — Injoinic **IP2366** (I²C version, LCSC C20415848)

One QFN-40 (5×5) does PD3.1 negotiation + buck-boost charge **and** discharge to
140W, explicit 2–6S, resistor-configurable (firmware optional). ~$0.85–1.51.
Rationale, alternatives (IP2368, TI BQ25756) and the trade study:
[`docs/research/charger-controller.md`](docs/research/charger-controller.md).

Open before layout: confirm in the IP2366 datasheet the max charge current at 6S
(~5.5A for 140W into 25.2V) and the reference application. Fallback if 6S current
or IP2366 sourcing falls short: TI BQ25756 + a USB-PD sink IC.

**⚠ Rework — cell balancing.** IP2366 charges the series stack as one; it has **no
per-cell balancing**, which a 2–6S Li-ion pack needs. Either spec internally-balanced
(BMS) packs, or add a balancing AFE (e.g. TI BQ769x0) + per-cell sense + balance
connector. Decision + options: [`docs/research/cell-balancing.md`](docs/research/cell-balancing.md).

## Agent tooling

This repo carries the incutec-eda agent skills (`.claude/skills/`) and uses the
`incutec-eda` CLI (installed from `~/OpenDrone/kicad-tooling/core`; the generic
tooling lives there and is reused, not copied). Capabilities:

- `component-research` skill → `incutec-eda research-report` — survey suppliers +
  datasheets + state-of-the-art, score, write a report into `docs/research/`.
- `incutec-eda source <Cxxxx> --project .` — import LCSC parts into project-local libs.
- `incutec-eda fields-audit/-set`, `place`, `sheets-scaffold/-ports` — repeating-step
  helpers on the schematic.
- `incutec-eda verify-sch/-pcb` — ERC/DRC gate (run after edits).

KiCad files are the source of truth; every agent change is a reviewable `git diff`.
See `.claude/skills/incutec-eda/SKILL.md` for the full command reference.
