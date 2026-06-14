# Architecture — block netlist (design intent)

Captured 2026-06-14 from design intent. **Open decision flagged in §"Discrete vs
AFE" — this netlist diverges from the scoped BQ76907 path.**

## Block netlist

```
PD brick (5–20V) → USB-C VBUS → IP2366 VBUS/VIO
USB-C CC1/CC2/DP/DM → IP2366 (PD negotiation)

IP2366 power stage  ← CRITICAL: transcribe from IP2366 datasheet Fig.11
  HG1/LG1/LX1/BST1 + HG2/LG2/LX2/BST2 → 4× H-bridge NMOS + L1 22µH + bootstrap
  CSP1/CSN1 = input sense 5mΩ ; CSP2/CSN2 = battery sense 5mΩ
  BAT → pack node (JST-XH main pins)
  config R: RBAT_NUM→6S, RPSET→PMAX, RVSET→4.2V/cell, RNTC→10k to GND
  I²C (GPIO19/20/1) → ESP32

Per-cell (×6, on JST-XH balance taps):
  cellN+ → divider → ADC ch
  cellN  → [bleed R + N-FET], gate ← ESP32 GPIO (level-shift on upper cells)

ESP32-C3: reads 6 cell V, drives 6 bleed FETs, IP2366 I²C, ESP-NOW radio
Manager variant: same board + SPI TFT
```

## Balancing (discrete passive, ≈300 mA)

- Bleed R: 15 Ω across each cell (≈0.28 A at 4.2 V), in series with a bleed FET.
- Bleed FET: logic-level N-MOSFET, low Rds(on).
- Cell sense: 1% resistor divider per tap scaled to <3.3 V (cell 6 ≈25 V → ~8:1).
- ADC: external I²C **ADS1115-class** (not ESP internal ADC — too nonlinear for
  4.2 V cell-limit enforcement).

## CRITICAL NOTES / open items

1. **Discrete vs AFE — decide before layout.** This netlist replaces the scoped
   **BQ76907** with discrete bleed + ADS1115. Consequence: **no hardware
   protection** (OV/UV/OC/SC/OT) — cell safety becomes firmware-only on the ESP.
   For Li-ion that needs an independent hardware backstop. Options:
   - (a) keep discrete (300 mA balance, full control) **+ add a standalone
     hardware protector** (e.g. a 2–6S secondary protector) for the safety backstop;
   - (b) **BQ76907**: hardware protect + 16-bit per-cell ADC + balancing in one
     $1.46 chip — but internal balancing is only ~35 mA (slow); add external FETs
     if 300 mA is required, driven by the BQ76907's cell-balance outputs;
   - (c) firmware-only protection — **not recommended for Li-ion.**
   Note: BQ76907 also eliminates items 3–5 below (it has referenced cell ADCs +
   balancing FETs built in).
2. **Bleed resistor wattage error.** 4.2 V / 15 Ω = 0.28 A → **P = 1.18 W**, which
   **exceeds a 1 W part**. Use 2 W (2512) with derating, or split (e.g. 2×30 Ω), or
   drop balance current. 1206 @ 1 W is not adequate.
3. **Bleed FET Vds.** Each bleed FET sits across **one cell (~4.2 V)**, not the
   stack — 30 V is fine/conservative, but the limiting issue is gate-drive
   *referencing*, not Vds (see 4).
4. **Upper-cell gate drive.** Correct: cells 3–6 have their source at up to ~21 V,
   so a GND-referenced 3.3 V GPIO can't switch them. Needs **per-cell high-side
   level-shift ×6** (NPN/PNP pair or dedicated high-side driver) — real parts +
   board area. This is the main cost of going discrete (an AFE integrates it).
5. **ADS1115 channel count.** ADS1115 = **4 single-ended channels**; 6 cells needs
   **2× ADS1115** (address-strapped) or an 8-ch ADC. Keep divided taps ≤3.3 V (the
   ~8:1 for cell 6 → ~3.1 V ✓). Use 0.1–1% resistors; high-value dividers to limit
   standing drain + match for 4.2 V accuracy.
6. **IP2366 power stage = datasheet Fig.11** — must transcribe the real HG/LG/LX/
   BST topology, FET selection, bootstrap, and sense placement from the datasheet.
   Blocked by the **NDA-only IP2366 docs** (see per-node-bom open risk); obtain first.

## Reconciliation

`research/per-node-bom.md` currently specs the **BQ76907** path. This netlist is the
**discrete** path. They are mutually exclusive on the BMS block — resolve item 1,
then update whichever BOM block is wrong so the repo has one source of truth.
