# Cell balancer / AFE — 2–6S decision (MUST include 2S)

**2026-06-14.** The IP2366 charges the whole stack and has **no per-cell sense or
balancing**. This node needs a separate analog front-end (AFE) that gives per-cell
voltage (for IR estimation) + passive balancing + per-cell protection, over I²C,
covering the **hard 2S-to-6S range**. The 2S floor is the killer constraint: the
common AFEs (BQ76952 3–16S, BQ769x0 ≥3S) do not reach 2S. This is the decision.
Tags: **FACT** = vendor datasheet/product page or LCSC; **UNVERIFIED** = inferred
or unconfirmed.

## Decision — TI **BQ76907** (2S–7S), single part covers the whole range

**FACT.** TI's BQ7690x family is the only mainstream AFE family that natively
spans 2S and reaches 6S with per-cell ADC, host-controlled passive balancing,
full protection, and I²C. Two SKUs cover it:

- **BQ76907 = 2S–7S** → covers 2S, 3S, 4S, 5S **and** 6S in one part. **Baseline.**
- **BQ76905 = 2S–5S** → cheaper drop-in *only if the product is capped at 5S*.
  Barely cheaper, so no reason to prefer it given the 6S requirement.

No part-split, no board variants. (ti.com BQ76907 = "2-series to 7-series";
BQ76905 = "2-series to 5-series".)

## Comparison table

| MPN | Mfr | Cell range | 2S? | Per-cell ADC | Balancing | Protection | I/F | LCSC | Verdict |
|---|---|---|---|---|---|---|---|---|---|
| **BQ76907RGRR** | TI | **2S–7S** | ✅ | 16-bit | internal-FET passive ~35 mA @4.2V | OV/UV/OCC/OCD/SCD + OT/UT | I²C 400k | C22458649, ~$1.46, stock ~992 | **CHOSEN** — covers full 2–6S |
| BQ76905RGRR | TI | 2S–5S | ✅ | 16-bit | same | same | I²C 400k | C22394458, ~$1.49, stock ~704 | Viable only for a 5S-capped SKU; fails 6S |
| MAX17320 | ADI | 2S–4S | ✅ | yes | internal-FET | yes | I²C | weak LCSC | REJECT — maxes at 4S; also a fuel gauge that overlaps IP2366 coulomb counting |
| BQ76952PFBR | TI | 3S–16S | ❌ | yes | host/auto | full | I²C/SPI | C2862742, ~$1.88 | REJECT — 3S min (needs top+bottom cell), no 2S |
| BQ76920 | TI | 3S–5S | ❌ | yes | external-FET | yes | I²C | — | REJECT — 3S min |
| ISL94202 | Renesas | 3S–8S | ❌ | 12-bit | external-FET | standalone | I²C | not on LCSC | REJECT — 3S min, weak sourcing |
| MC33772C | NXP | 3S–6S | ❌ | yes | internal | full | SPI/daisy | automotive | REJECT — 3S min, SPI/daisy, oversized |

**FACT** that 2S is impossible for the rejected TI/Renesas/NXP parts: their
datasheets are titled "3-Series to N-Series" and require top+bottom cell taps.
Cheap Chinese protection ICs (S-8209B, DW01, LC05732, S-82xx) are fixed-cell
protection-only, **no host I²C telemetry** → not substitutes.

## Why BQ76907

- **Only single part that natively covers 2S through 6S** (to 7S) with
  sense + balance + protection over I²C. **FACT.**
- **Balancing:** passive, host-controlled, **internal on-chip FETs** (RDS(on)
  ~80 Ω typ). With the recommended ~20 Ω per-cell series input-filter R this gives
  ~35 mA balance at 4.2 V/cell — adequate for trickle top-balancing FPV packs that
  are charged frequently. Expandable with external FETs/BJTs if faster balance is
  wanted. (TI app note SLUAAP7; balance current **UNVERIFIED** to the milliamp —
  confirm against the BQ76907 datasheet electrical table.)
- **Protection:** OV, UV, OCC/OCD, SCD, OT/UT via thermistor; integrated low-side
  CHG/DSG NFET drivers; programmable LDO; coulomb counter; 400 kHz I²C w/ optional
  CRC. **FACT.**
- **Package/cost:** VQFN-20-EP 3.5×3.5 mm, ~$1.46. The balancing+protection+sense
  subsystem BOM is dominated by the ~$1.50 IC. **FACT.**

## How it pairs with IP2366

Both are I²C, so the ESP32 host bridges them on the bus (assign distinct
addresses — IP2366 is 0x75; BQ76907 per its datasheet). Clean division of labor:

- **IP2366** — PD sink + buck-boost charge/discharge + **pack-level** V/I telemetry
  + charge-current setpoint. (No per-cell anything.)
- **BQ76907** — **per-cell** V (the data the IR algorithm needs), passive
  balancing, and per-cell OV/UV/OT protection with its own CHG/DSG FET drivers.

The IR→current algorithm reads ΔV/ΔI **per cell** from the BQ76907 ADC and ΔV/ΔI
**pack** from the IP2366 ADC, estimates IR, and writes the IP2366 charge setpoint —
bounded by C-rate, thermal, and the 3 A connector ceiling.

## Sourcing

- **BQ76907RGRR** — LCSC **C22458649**, ~$1.46/ea, stock ~992, lifecycle ACTIVE.
  Also DigiKey ~$0.87–1.79. **FACT.**
- BQ76905RGRR (5S-cap alt) — LCSC **C22394458**, ~$1.49/ea, stock ~704, ACTIVE.
- Stock ~700–1000 is fine for prototyping; soft volume risk → JLCPCB consignment
  covers it. No EOL/NRND flag on either.

## Open / UNVERIFIED

- Exact internal balance current vs the 20 Ω filter R (~35 mA @4.2V inferred from
  SLUAAP7) — confirm against the BQ76907 datasheet before finalizing balance timing.
  If faster balance is wanted, plan external balance FETs.
- Verify the cell-input shorting scheme for running 6 of 7 channels at 6S (6S is
  mid-range, high confidence fine, but check the pin-config table).
- No LCSC-native Chinese AFE found that does 2–6S + per-cell I²C ADC + balancing;
  BQ76907 stands as the recommendation.

## Sources

- TI BQ76907 — https://www.ti.com/product/BQ76907
- TI BQ76905 — https://www.ti.com/product/BQ76905
- TI BQ76952 datasheet — https://www.ti.com/lit/ds/symlink/bq76952.pdf
- ADI MAX17320 — https://www.analog.com/en/products/max17320.html
- Renesas ISL94202 — https://www.renesas.com/en/products/isl94202
- NXP MC33772C — https://www.nxp.com/products/MC33772C
