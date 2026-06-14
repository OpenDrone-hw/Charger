# Charger

Open-source distributed USB-C charging system for FPV packs: off-the-shelf USB-C PD
bricks feed small smart **charging nodes** that take USB-C in and output **2–6S on
JST-XH**, auto-pick charge current from each pack's **internal resistance**, balance
the cells, and report over a wireless mesh. An optional **manager node** has a screen
and shows nearby nodes. Designed in KiCad for JLCPCB. Part of the incutec OpenDrone line.

> Status: **early — architecture locked, schematic in progress.** The KiCad project is
> scaffolded in `hardware/`; pin-level wiring, the IP2366 power stage, and layout are WIP.

## Architecture — three ICs per node

| IC | Role | Owns |
|---|---|---|
| **IP2366** (Injoinic, QFN-40 5×5) | USB-PD 3.1 sink + single-inductor buck-boost charge/discharge | **Power** |
| **BQ76907** (TI, 2–7S, QFN-20) | cell monitor + protector + balancing | **Safety** (hardware) |
| **ESP32-C3** | supervisor + IR→current loop + ESP-NOW radio | **Intelligence** |

```
USB-C VBUS ─► IP2366 (PD + buck-boost) ─BAT+─► Pack+ (JST-XH)
                  │                              │
                  │                          [pack cells] ◄─ VC0..VC6 ─► BQ76907
                  │                              │
   IP2366 GND ◄───┴─[CHG][DSG FETs]◄─[sense R]◄─ Pack- (JST-XH)
                          ▲ driven by BQ76907 ──┘
ESP32-C3 ── I²C (master) ── IP2366 + BQ76907      TPS560430 buck (VBUS|BAT) → 3V3 → ESP + BQ
```

All charge current returns through BQ76907's CHG/DSG FETs, so a per-cell OV/OC/OT
fault cuts charging **in hardware**, independent of firmware. One ESP-mastered I²C
bus carries control; a 3.3 V buck (from VBUS or the pack) powers the ESP + BMS logic.
(Full trade studies that led here live in git history.)

## Specifications

| | |
|---|---|
| **Input** | USB-C, USB-PD (off-the-shelf brick; 5–20 V) — we don't design the PD source |
| **Battery** | 2–6S Li-ion / LiPo / LiFePO₄ (8.4–25.2 V). **2S minimum** (no 1S) |
| **Charge current** | ≤3 A (JST-XH limit) → **~65 W at 6S**; scales with cell count (2S ≈ 25 W) |
| **Charge-current selection** | automatic, from measured per-cell internal resistance |
| **Balancing** | BQ76907 host-controlled, internal (~tens of mA — sufficient for small FPV packs) |
| **Protection** | BQ76907 hardware OV/UV/OC/SC/OT via external CHG/DSG NFETs |
| **Sensing** | 16-bit per-cell ADC (4 mV) + coulomb counter → IR + SoC |
| **Connector** | JST-XH (combined charge + balance, ≤3 A) |
| **Wireless** | ESP32-C3 (Wi-Fi/BLE, ESP-NOW mesh); optional manager node + SPI TFT |
| **Dissipation** | ~5–7 W at full 6S/65 W (dominated by the IP2366 stage) |

## Key components

| Block | Part | LCSC | Status |
|---|---|---|---|
| Power stage | IP2366 (PD3.1 + buck-boost, QFN-40) | C20415848 | sourced |
| BMS | BQ76907 (2–7S monitor/protect/balance) | C22458649 | sourced |
| MCU / radio | ESP32-C3-MINI-1 | — | to source |
| 3.3 V rail | TPS560430 buck (VBUS/BAT → 3V3) | — | to source |
| USB-C input | receptacle + USBLC6 ESD + VBUS TVS | — | to source |
| Protection FETs | CHG + DSG low-side N-MOSFET ×2 | — | to source |
| Output | JST-XH (B7B-XH-A) + ~4 A fuse | — | to source |
| Manager delta | SSD1306 OLED / small SPI TFT | — | to source |
| Power-stage support | 4× H-bridge NFET, 22 µH inductor, 2× 5 mΩ sense, config R, caps | — | to source |

No single IC does PD + 2–6S buck-boost charge + per-cell balancing (verified; closest
is Renesas RAA489118, still no balancing). The IP2366 + BQ76907 + ESP split is the
standard architecture, not a compromise.

## Charge current from internal resistance

The ESP steps IP2366's charge current, reads ΔV/ΔI per cell (BQ76907 ADC) and pack
(coulomb counter), computes per-cell IR, then sets the charge current to
`min(per-cell limit, 3 A connector cap, thermal)`. Big healthy 6S packs charge near
3 A; tiny/high-IR whoop packs are auto-throttled. Balancing runs during charge.

## Open items / risk

- **IP2366 I²C register map + power-stage reference (datasheet Fig.11) are NDA-only** —
  obtain from Injoinic and bench-verify before the power stage and the IR→current loop.
  Fallback power stage: Renesas RAA489118.
- Balance current is internal-only (~tens of mA); confirm sufficient, else evaluate
  external balance FETs.
- Variable 2–6S: 7 VC taps wired; firmware detects populated cells.
- "to source" parts above need LCSC selection.

## Repository

- `hardware/` — KiCad project: `Charger.kicad_sch` (root) → `power` / `bms` / `mcu`
  hierarchical sheets; project-local libraries in `hardware/libs/`.
- `CLAUDE.md` — agent/working instructions and tooling.

Hardware: CERN-OHL-S. Open source.
