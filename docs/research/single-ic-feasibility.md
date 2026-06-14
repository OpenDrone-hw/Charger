# Single-IC feasibility — "one chip that does everything for 2–6S"

**2026-06-14.** Question: is there a *single* IC that integrates USB-PD sink +
2–6S buck-boost charge + per-cell balancing/sense, so we could drop the IP2366 +
separate-AFE two-chip plan? This documents the verdict, the evidence, and the
closest options. Tags: **FACT** = sourced from a vendor datasheet/product page;
**UNVERIFIED** = engineering inference or unconfirmed sourcing.

## Verdict — NO

**FACT.** No IC on the market integrates all three of USB-PD sink + multi-cell
(2–6S) buck-boost charge + on-chip per-cell balancing/sense. The market splits
into two non-overlapping camps and neither crosses the gap. Checked across TI,
ADI/Maxim, MPS, Renesas, Southchip, Injoinic, Infineon/Cypress.

- **Camp A — PD + multi-cell buck-boost, NO balancing.** TI BQ25790/92/98/703A/720
  (1–4S), MPS MP2651/MP2652 (1–4S / 2–5S), MPS MP2762A (2S), Renesas
  RAA489xxx (up to 2–7S), ADI MAX77962 (2S). All sink USB-C/PD and do buck-boost
  charging, but **none balance on-chip** — their own app notes delegate balancing
  to a separate gauge/AFE. **FACT.**
- **Camp B — integrated balancing.** TI BQ25887 / BQ25882 are the only mainstream
  chargers with on-chip per-cell balancing FETs (~400 mA). But they are
  **2S-only, boost-only (no buck/buck-boost), and have NO PD sink** (plain
  3.9–6.2 V USB in). Satisfy 2S, cannot scale to 6S, cannot sink PD. **FACT.**

**Root cause (UNVERIFIED — engineering inference, consistent with all datasheets
reviewed).** Per-cell balancing needs individual taps to every cell node (the
BQ769xx AFE function). High-power buck-boost charger silicon and high-voltage
multi-cell sense AFE silicon use different process/pin architectures, so vendors
deliberately split them: charger IC + separate AFE/gauge. This makes the chosen
**IP2366 + external 2–6S AFE** the industry-standard architecture, not a
compromise.

## Closest options (all rejected as a single-IC solution)

| MPN | Mfr | What it has | What it's missing | Verdict |
|---|---|---|---|---|
| **RAA489118** | Renesas | PD-EPR sink + buck-boost, **2–7S** (covers full range), SMBus, 4×4 QFN | **No per-cell sense, no balancing** (pack-level only) | REJECT — offers nothing IP2366 lacks except slightly wider cell range; misses balancing equally |
| **BQ25887** | TI | Only mainstream charger with **integrated balancing** (~400 mA FETs), I²C, ADC | 2S-only, boost-only, **no PD sink** | REJECT — fails 2–6S and PD |
| **BQ25790/98** | TI | PD sink + buck-boost, 5A, I²C | 1–4S (fails 6S), no balancing | REJECT |
| **MP2762A** | MPS | PD buck-boost, 6A, integrated FETs, I²C | 2S-only, no balancing | REJECT |
| **SC8815** | Southchip | 1–6S buck-boost controller, I²C, ADC, bidirectional | **External FETs, no PD sink, no balancing** | REJECT — same class as IP2366 but worse |
| **MAX17330/32** | ADI | Per-cell charger + gauge + protect + **balance** | **1S only** (needs 6 ICs for 6S), no PD; MAX17330 NRND | REJECT — opposite of a single-IC solution |

## Implication

The IP2366 (PD3.1 sink + 2–6S buck-boost, no balancing) is already best-in-class
as the single power chip. A separate balancing/sense path is unavoidable. The
hard 2S requirement also kills the only balancing-integrated chargers (BQ2588x
are 2S-only *and* boost-only/no-PD anyway). Proceed with the two-chip plan; the
AFE decision is in [`cell-balancer.md`](cell-balancer.md).

## Sources

- Renesas RAA489118 — https://www.renesas.com/en/products/power-management/battery-management/battery-charger-ics/raa489118-buck-boost-battery-charger-smbus-interface-general-30v-and-usb-pd-epr
- TI BQ25887 — https://www.ti.com/product/BQ25887
- TI BQ25798 datasheet — https://www.ti.com/lit/ds/symlink/bq25798.pdf
- MPS MP2762A — https://www.monolithicpower.com/en/mp2762a.html
- Southchip SC8815 — https://www.southchip.com/en/product/SC8815
- ADI MAX17330 datasheet — https://www.analog.com/media/en/technical-documentation/data-sheets/MAX17330.pdf

## Open / UNVERIFIED

- Live LCSC/JLCPCB stock+price for RAA489118 / BQ25887 not confirmed (moot — both
  rejected on architecture, not sourcing).
- Did not exhaustively sweep obscure/new-2025 Chinese SKUs (Injoinic full catalog,
  Cellwise, Sino Wealth) in Chinese-language datasheets. Confidence still high that
  none combines all three, given the process-architecture reason.
- RAA489118 full datasheet PDF not opened to triple-check for a hidden
  balancing-tap feature; product brief strongly indicates pack-level only.
